# Assessment: applying DBT to Hercules

2026-08-19. Grounded in [hercules-internals.md](hercules-internals.md) and
[donors.md](donors.md). Everything here labeled *projected* is exactly that —
no Hercules DBT code exists.

## 1. The question, and the shape of the answer

Hercules is an op-code interpreter with decode already squeezed flat (one
64K-entry table lookup per instruction). The in-house DBT portfolio proves out
the techniques at 0.6–9 BIPS on four guests. Do they transfer?

Yes — but not by porting the riscv/z80 "translate everything" model. Those
translators earn their BIPS from invariants z/Architecture denies: fixed
instruction formats, no MMU, no interrupts, no architectural flags, unchecked
one-instruction memory access, code that is never written. The transferable
architecture is **VCC's hybrid**: the JIT is an additive accelerator over the
existing, unchanged interpreter, which remains the semantic authority and the
fallback for everything hard.

Hercules is unusually well-suited to that model:

- ~1,300 handlers with the uniform signature `(BYTE* inst, REGS* regs)` are
  ready-made trampoline targets — VCC's level-1 tier maps directly.
- The interpreter's own fast paths are already gated on the predicates that
  would gate JIT entry (`SuccessfulBranch` bails on `permode` / `execflag`).
- A golden reference interpreter with decades of trust is the prerequisite for
  the lockstep-verifier methodology all four donors share.

## 2. The load-bearing design decision: guest state stays in REGS

Hercules delivers program checks by `longjmp(regs->progjmp)` out of the
handler. A longjmp through JIT frames restores callee-saved host registers to
setjmp-time values — dirty guest state cached in host registers at a faulting
point is silently lost, and nearly every z instruction can fault.

VCC's negative result dissolves the problem: registerizing guest state
measured as noise on modern out-of-order cores. So:

- Pin one host register at `&REGS` (plus mainstor base, block-cache base).
- GPRs, PSW, CC stay memory-resident in REGS; inline bodies do
  `ldr/str [x_regs, #offsetof(...)]`.
- Set `regs->ip` before anything that can fault (one store — VCC paid the
  same and it was in the noise).

Then **precise interruptible state falls out for free**, longjmp included; the
existing `program_interrupt` path works unmodified; and the lockstep verifier
stays trivial (state to compare is always in REGS). The JIT's wins come from
killing dispatch, re-decode, dead CC computation, and the `maddr_l` gauntlet —
not from register allocation.

## 3. The subsystem plan

### Condition code — z80's static liveness, simplified
Backward pass over the block; the lattice is a single "is CC live here" bit
(vs z80's 8). Most z arithmetic sets CC and few instructions consume it, so
most CC computation is dead and never emitted. Carry over the two soundness
rules (live-out, not live∩write; quirky-input ops read-all) and VCC's
`CC_UNKNOWN`-means-called-handler lockstep guard. `BC`/`BRC` become a load of
`psw.cc` + mask test + branch; mask 15/0 folds to unconditional/nop at
translation time.

### Memory — inline the hit path of the existing TLB
No one-instruction loads under DAT, keys, and AR mode. But `regs->tlb` already
exists; the JIT inlines a slimmed hit probe (~8–12 host instructions: index,
tag compare, access-type check, host-address compute + byte-swap) and calls
`maddr_l` on miss — which handles keys, ref/change bits, protection, AR mode,
and faults exactly as today. slow-32's measured 2–8% cost for inline
guards+W^X says the probe overhead is bearable. Big-endian: every access swaps
(`rev`/`movbe`) — pervasive, cheap.

### Invalidation — VCC's generation machinery + z80's span gating
- Blocks keyed by virtual PC and arch/address-space context; **linking only
  indirectly through the live cache slot** (VCC's chain stub) so invalidation
  never unpatches anything.
- DAT-affecting events (LCTL, PTLB, IPTE/IDTE, PSW swaps, SSKE...) →
  generation bump, O(1). z/OS-lineage systems hammer these like OS-9 hammered
  the GIME, so **incumbent survival** (re-validated blocks keep their thunks)
  is mandatory, not optional.
- SMC: page-granular dirty bitmap over mainstor pages that hold translations
  (1 bit per 4K page = 32KB/GB), 3-instruction store-side check, span-gated
  precise kill. Channel I/O writes funnel through the same hook. VCC's
  "block invalidates its own tail" case will occur; the fix is on file.

### Interrupts — ChainBreak
A maintained one-byte "would the dispatcher act now" flag: pending I/O /
external / timer interrupts pre-masked against the PSW system mask and CR
masks, recomputed at mutation sites (Hercules's IC-bit machinery already has
the inputs). Tested once per block/chain hop; loop back-edges inside traces
test it too. Interrupt latency is bounded by max block length, as in VCC.

### Dispatch and cache
slow-32's hashed, tagged block cache (z PCs are 64-bit, halfword-aligned;
1:1 mapping is not available). Per-CPU code caches — translations are already
per-arch-mode, and per-CPU sidesteps most JIT-metadata locking.

### Permanently interpreted
`EX`/`EXRL` (runtime-materialized instructions), PER mode, active TXF
transactions, SIE, all privileged/control ops, crypto/vector/DFP, and any
SS-format storage op not worth inlining. Rare-path cost is one trampoline call
into the existing handler — the hybrid's whole point.

## 4. The three genuinely new problems

1. **SMP.** All four donors are single-threaded. Guest semantics are covered
   by Hercules's existing atomics/mainlock; the new problem is JIT metadata
   coherence — cross-CPU store-into-translated-code, IPTE/SSKE broadcast vs
   per-CPU caches, W^X bracketing per thread. Hardest single design problem.
2. **Breadth.** ~1,300 handlers, but dynamic frequency is brutally
   concentrated (L, ST, LA, BC/BRC, TM, LR, MVC, CLC, LTR, AHI, LG/STG...).
   Level-2 inlining of the measured top ~40 with CC-DSE should capture most of
   the win; everything else stays a trampoline forever.
3. **Variable-length instructions** (2/4/6 bytes, halfword-aligned) — affects
   block hashing, span accounting, and the pre-decode struct, all solved
   shapes in the donors, just not the cheap versions.

## 5. Prior art

- **QEMU TCG s390x** — existence proof that z DBT works; roughly 1-BIPS-class
  in slow-32's own cross-benchmarks. Lacks the hybrid's near-zero fallback
  cost and CC-DSE; doesn't run the classic-OS workloads Hercules does.
- **IBM zPDT/ZD&T** — commercial proof on x86 hosts.
- **Hyperion itself** — zero prior JIT work; this would be a first.

## 6. Build order (the de-risked ladder)

0. **Measure first** (an afternoon, no fork needed): run hyperion's
   instruction-frequency counters over a real target workload — MVS 3.8j
   batch and/or a Linux/390 build — to get the dynamic opcode mix. This sizes
   the inline set, validates the projection, and forces the first scoping
   choice: S/370 mode (where hobbyist CPU-bound work lives) vs z/Arch mode
   (Linux, modern). `ARCH_DEP` triple-compilation means you pick one first
   and the others keep interpreting.
1. **Verifier before emitter** (the unanimous donor lesson): per-block
   lockstep against the unchanged interpreter, with instruction-count parity
   as a day-one invariant. z80's `-V` design; Hercules interpreter is the
   oracle.
2. **VCC's ladder**, each rung shippable: pre-decode struct → block cache +
   recording (validated against real execution) → level-1 trampolines
   (expect ~zero speedup — it exists to prove every correctness invariant:
   interrupt boundaries, invalidation, SMC, longjmp) → level-2 inlining of
   the measured top-N with CC liveness → chaining via chain stub → traces.
3. **AArch64 backend first** (primary dev machines; z80 and vcc are a64-only
   donors), x86-64 second via the riscv/slow-32 emitters. Null backend as a
   first-class configuration for Hercules's platform spread.
4. Adopt the donor tooling wholesale: A/B env-var kill switches, rank
   bisection of the inline set, write-stream diffing, boot-a-real-OS smoke
   suite (IPL MVS 3.8j / TK5, run a job, compare output).

## 7. Expected performance — projection, clearly labeled

Per slow-32's rule, a BIPS figure without a machine and a date is a rumor with
units; these are *estimates* against the reasoning above, on Apple-Silicon-
class hosts:

- Hot problem-state code: **3–8× over the interpreter**. Per-instruction
  inline bodies (5–8 host insns for register ops with dead CC; ~10–15 for
  loads/stores via the inline TLB probe) vs the interpreter's ~20–60.
  Interpreter sustains a few hundred MIPS/CPU → low-BIPS territory.
- Supervisor-heavy and I/O-bound work: **→ 1×** (dominated by `maddr_l` slow
  paths, channel emulation, and interpreted privileged ops). riscv's
  realistic-workload data (1.0–4.6× vs qemu-user) is the honest calibration.
- The memory path is the ceiling. Nothing recovers riscv/z80's one-instruction
  loads under DAT + keys; the win is capped by how thin the inline TLB hit
  path can be made.

## 8. Repo strategy

The DBT requires forking SDL-Hercules-390/hyperion (the work is inside
`cpu.c`/`opcode.h`'s orbit, not wrappable from outside; though the backend
itself should live in cleanly separated new files per VCC's seam). Current
thinking: docs stay in `~/hercules/docs/dbt/` until the work starts, then move
to a dedicated ~-level project holding the fork, keeping this repo as the
Docker packaging it is today.
