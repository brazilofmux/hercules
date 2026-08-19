# Hyperion execution core — recon notes

Surveyed at SDL-Hercules-390/hyperion commit `ae9b497` (2025-12-07). This is
the machinery a DBT would displace, and the interface it must honor.

## Dispatch loop

Everything is triple-compiled per architecture via `ARCH_DEP` (S/370, ESA/390,
z/Arch), giving `s370_run_cpu` / `s390_run_cpu` / `z900_run_cpu` selected
through a jump table (`cpu.c:2310`).

The inner loop (`fastest_no_txf_loop`, `cpu.c:2088`) is a table-threaded
interpreter:

```c
(_oct)[ fetch_hw( _ip )]( _ip, _regs );     /* EXECUTE_INSTRUCTION, opcode.h:1491 */
```

- One flat runtime opcode table per REGS (`regs->ARCH_DEP(runtime_opcode_xxxx)`),
  indexed by the **entire first halfword** — the "rolled-out tables". Two-byte
  opcodes resolve in a single lookup; decode is as cheap as an interpreter gets.
- Unrolled ×2, `MAX_CPU_LOOPS = 256` (`hconsts.h:36`); interrupt poll
  (`INTERRUPT_PENDING` → `process_interrupt`) once per 256 instructions.
- When the TXF facility is enabled there are two additional, slower loop
  variants (`txf_facility_loop`, `txf_slower_loop`) checking `regs->txf_tnd`
  per instruction (`cpu.c:2115+`).

**Per-instruction cost that remains after all this tuning:** one indirect call
through the table, operand re-decode inside the handler (RR/RX/RIL/... macros),
eager condition-code computation, and `regs->ip` bookkeeping. That is the
DBT's decode/dispatch budget.

## Instruction fetch and branches — the AIA

Instruction fetch pays DAT **once per page**: `regs->aip` (mapped page),
`regs->AIV` (its virtual address), `regs->aie` (end sentinel; also the
`UNROLLED_EXECUTE` loop-break test, `opcode.h:1519`).

`SuccessfulBranch` (`cpu.c`, ARCH_DEP) is the block model in embryo:

- Same-page branch target, `!regs->permode && !regs->execflag` → just move
  `regs->ip` within the mapped page. Fast path.
- Cross-page → set `psw.IA`, invalidate `aie`, force a fresh `instfetch`.
- `SET_BEAR_REG` maintains the breaking-event address on every taken branch.

Note the gates: PER mode and EX in flight already disqualify the interpreter's
own fast path. The same predicates gate JIT entry for free.

## Data path — `maddr_l`, the real cost center

Every operand access funnels through `MADDR`/`MADDRL` (`feature.h:995`) →
`ARCH_DEP(maddr_l)` (`dat.h:~305`), a software TLB probe:

```
tlbix = TLBIX(addr);
hit requires ALL of:
  - AEA control register matches TLB entry's ASD (or common-segment bit)
  - access key is zero or matches cached storage key
  - (addr & TLBID_PAGEMASK) | regs->tlbID  ==  TLB_VADDR(tlbix)
  - acctype compatible with cached access rights
miss → full DAT walk ("many, many instructions", per the comment)
```

Plus storage-key reference/change updates, fetch/store protection, AR-mode
resolution, SIE guest translation. **Dozens of host instructions per guest
load/store on a TLB hit.** This — not decode — is where the interpreter's
cycles go, and it bounds what any DBT can achieve (no z translator gets the
riscv/z80 one-instruction memory access).

The good news: `regs->tlb` is already the software TLB a JIT wants to inline
the hit path of. The slow path (`maddr_l` itself, then the DAT walk) is
battle-tested and callable as-is.

## Program checks — longjmp

`regs->program_interrupt(...)` delivers program exceptions by
`longjmp(regs->progjmp)` out of the handler, unwinding to `run_cpu`. Handlers
assume `regs->ip` points at the current instruction (ILC derivation, PSW
backup). Two consequences for a DBT:

1. Any JIT design that caches dirty guest state in host callee-saved registers
   loses it when a helper longjmps. (See assessment.md — this argues for
   memory-resident guest state.)
2. `regs->ip` must be correct before every potentially-faulting operation,
   which is nearly every z instruction.

## EX / EXRL

`DEF_INST(execute)` (`general1.c:5723`): fetches the target instruction at
runtime, ORs byte 1 with R1's low byte into `regs->exinst`, sets
`regs->execflag`, backs `regs->ip` up by the target ILC, and dispatches through
the runtime opcode table. Recursive EX is a program check. A
runtime-materialized instruction — the canonical stay-interpreted op.

## Condition code

`regs->psw.cc` is a byte, computed **eagerly** by nearly every arithmetic/
logical/compare handler, consumed by `BC`/`BRC`/`BCR` mask tests. No laziness
anywhere. This is the z80-style dead-CC-elimination opportunity: most CC
writes are dead.

## SMP model

- One host thread per guest CPU (`cpu_thread` → `run_cpu` jump table).
- Interlocked storage ops via host atomics (`hatomic.h`), with
  `OBTAIN_MAINLOCK` (`hmacros.h:438`) as the interlocked-update fallback.
- Cross-CPU TLB invalidation (IPTE/IDTE/PTLB/CSP) is already broadcast through
  existing mechanisms; a JIT's translation caches would need to ride the same
  events.

## Instruction inventory

~1,300 `DEF_INST` bodies (grep at `ae9b497`), spread over `general1/2/3.c`,
`esame.c`, `control.c`, `float.c`/`ieee.c`/`dfp.c`, `zvector*.c`, `crypto`,
etc. `OPTION_OPTINST` (`featall.h:90`) provides specialized pre-decoded
variants of hot opcodes — evidence the maintainers already chased dispatch
cost as far as an interpreter allows.

Instruction-frequency counting exists in-tree (`BEG_COUNT_INSTR` /
`END_COUNT_INSTR` in `EXECUTE_INSTRUCTION`, `icount` machinery) — this is the
tool for sizing a JIT inline set against a real workload before writing any
emitter.

## Prior art in-tree

None. No JIT/dynarec/translation mentions in `_TODO.txt` or `CHANGES`.
