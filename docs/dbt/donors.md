# Donor projects — what transfers, what doesn't

Four in-house DBT codebases surveyed 2026-08-19. Ordered by relevance of their
*architecture* to a Hercules DBT, not by BIPS.

Cross-cutting note on the shared encoder: `emit_a64.h` has traveled
riscv → z80 → vcc. **Take VCC's copy** — it fixes a shift-by-32 UB bug in the
logical-immediate encoder (`a64_encode_logical_imm32` when `r==0`) that, per
vcc commit `f8360ff`, still exists in the riscv and z80 copies.

---

## ~/vcc — the integration story (6809/6309 CoCo3, OS-9 Level II)

**Why it matters most:** it is the only donor that bolted a JIT onto a mature,
trusted interpreter — which is exactly the Hercules problem. The contract
(`BlockJit.h:30-38`): *"JIT is purely additive: when a block has a
`native_entry`, the dispatch loop calls it; otherwise it falls back to the
existing interpreter loop. There's no flag day."*

**Architecture.** Four-tier dispatch loop (`hd6309.cpp:7871`): native thunk →
pre-decoded replay → partial replay → single-step-and-record. Blocks are formed
by **recording** — the interpreter single-steps and each instruction's actual
next-PC is validated before commit, so block contents are validated against
real execution, not against a possibly-divergent decoder. Level-1 JIT =
trampolines calling the unchanged interpreter handlers with pre-decoded
operands; level-2 = selective inlining of the top-N handlers. The JIT is never
asked to be complete. Backend seam is a 6-function interface with a null
backend as a first-class build config.

**Ideas to steal wholesale:**

- **Chain-stub linking through the live cache slot** (`BlockJitA64.cpp:1391`):
  one shared stub per arena epoch re-checks budget / pending-interrupt byte /
  slot tag / generation / cycle fit, then `br` into the next thunk. **No
  patching anywhere** — clearing `native_entry` or bumping the generation
  severs every inbound link with zero unlink bookkeeping. Measured +19%.
- **Generation-counter bulk invalidation + incumbent survival**
  (`BlockCache.h:632`): MMU remap → `generation_++` (O(1)); hot code that
  re-records identically is re-validated by memcmp and **keeps its thunk and
  hotness**. Commit `f8360ff`: *"without this the JIT evaporated after boot"*
  (OS-9 remaps on every process switch — the DAT analogue). MMU setters are
  idempotent-first: unchanged writes skip invalidation entirely.
- **`ChainBreak`** (`hd6309.cpp:199`): a maintained one-byte "would the
  dispatcher act right now" flag — pending lines **pre-masked against CC/PSW
  masks**, recomputed at every mutation site. Replaced testing raw pending
  state, which broke on a permanently-asserted-but-masked line.
- **Three-level write invalidation:** 32-byte coarse page bitmap (rejects
  ~99.99% of stores) → O(1) reverse map → generation. Plus the three
  hard-won SMC fixes, especially: *a block can invalidate its own later bytes
  mid-execution* (`ReplayBlock` re-checks its own generation; real case was
  BASIC's CHRGET rewriting its own operand).
- **Budget slack + cycle drift** (`kBudgetSlack`, `CycleDrift`): decouple
  block granularity from timing-slice granularity without losing average
  timing. Worth +61% there.
- **The verification toolkit:** snapshot/run/restore/replay differential
  (`VCC_VERIFY_PURE`), rank-bisection of the inline set (`VCC_INLINE_MAX` —
  how a real emitter bug was found), write-stream diffing, and a smoke suite
  that boots a real OS (NitrOS-9) per change.
- **Negative result:** registerizing guest state measured as noise — "load-
  store to a hot struct is nearly free on a modern OoO core"
  (`docs/porting-macos.md:197`). Guest state lives in one struct behind a
  pinned base register. This is load-bearing for Hercules (see assessment).

**Doesn't transfer:** every 16-bit-address structure (`uint16_t` reverse map
over the whole address space, `pc & 0xFFF` cache index, 4096×256B slots),
`MAX_BLOCK_INSNS=14`, and the single-threaded assumptions (global cpu_state,
non-atomic stats, one code cache).

**Numbers:** ~3203× realtime on DECB ≈ 2.9 GHz effective 6309 ≈ ~570–640 MIPS
(derived; repo reports realtime multiples, not MIPS). ~14× over its own
pre-JIT block-cache baseline. Read `docs/porting-macos.md` end to end — it is
design doc, results log, and failure log in one.

---

## ~/z80 — the condition-code playbook (CP/M 3, AArch64 host only)

**Why it matters:** Z80's flags problem is the closest analogue to z/Arch's CC,
and the solution shipped is simple and fast.

**The technique: static per-bit dead-flag elimination.** No lazy-flag runtime
descriptors (tried, abandoned — vestigial fields remain), no host-NZCV
harvesting. A 16-line backward liveness pass (`dbt_a64.c:2007`):

```c
uint8_t live = 0xFF;
for (int i = n_ops - 1; i >= 0; i--) {
    op_flag_effects(&decs[i], &rd, &wr);
    fmask[i] = live;                    /* LIVE-OUT mask for op i */
    live = (live & ~wr) | rd;
}
```

Emitters gate each flag chunk on `fmask`. `ADD` before another `ADD` emits no
flag code at all. Two soundness rules (learned the hard way, `CLAUDE.md:88`):
`fmask` is **live-out**, not live∩write (pass-through ops must see bits outside
their write set); ops with quirky flag *inputs* classify as read-all.

**For z/Arch this transfers and gets simpler:** the CC is one 2-bit value, so
the lattice collapses to a single "is CC live here" bit, and `BC`/`BRC` mask
tests map to the same test-and-branch pattern.

**Other assets:**

- **SMC handling that actually works**: per-guest-byte code bitmap, checked in
  3 inline instructions on the store fast path; **span-gated invalidation**
  (each cache entry records its exact guest byte extent; a store kills only
  genuinely-covering blocks). Span gating alone took zexdoc 1.99 → 3.25 BIPS.
  Bitmap is never cleared — false positives are cheap, a false negative was a
  real stale-translation bug. Survived 7.4M invalidations in one zexdoc run.
  (Byte-granular bitmap doesn't scale to GB guests; page-granular does. The
  span-gating insight survives.)
- **Static register pinning** of all guest regs in callee-saved host regs,
  live *across* block chains. Works because the Z80 hot set fits; z/Arch's
  won't — but combined with vcc's negative result above, that's fine.
- **Lockstep verifier `-V`**: per-block (not per-instruction) — assert shadow
  matches, run JIT block, step shadow interpreter exactly `jit_insns` times,
  compare all 21 registers **and all 64KB of memory**. Depends on a hard
  **instruction-count-parity invariant** between JIT and interpreter (e.g.
  LDIR counts as 1 in both). Cheap enough to verify a 5.76B-instruction run
  in 46s. This invariant is worth adopting from day one.
- **Negative result:** a shadow return-address stack for CALL/RET measured a
  consistent 3–6% *loss* — Apple Silicon's indirect predictor already handles
  the inline probe (postmortem at `dbt_a64.c:485`).
- Interrupts: **entirely unimplemented** — nothing to copy, but the substrate
  (precise state at every block boundary, exact pinned instruction counter) is
  the right discipline.

**Numbers (re-verified on disk 2026-08-19, M5 Max):** SQUARO 4.31 BIPS,
zexdoc 3.22 BIPS, interpreter-fallback rates 0.016% / 0.199%. ~9.1 kLOC total.

---

## ~/slow-32 — the scaffolding and the process (custom ISA)

**Why it matters:** the most reusable non-translator assets, and the port
process itself is documented as a playbook.

- **`tools/dbt/block_cache.{h,c}`** — near drop-in: 128K-entry open-addressed
  hash + 64K-entry direct-mapped compact table probed inline by JIT code,
  chaining with an O(1) reverse index (newly translated block patches all
  waiting predecessors), flush-on-full with headroom check, `MAP_JIT`/W^X
  bracketing, hand-rolled AArch64 icache flush. Needs only halfword-aligned
  hashing and 64-bit tags for z.
- **Staged delivery specs** — `docs/dbt/STAGE{1..4}-SPEC.md`: Stage 1 =
  working translator with no cache; Stage 2 = cache + direct chaining;
  Stage 3 = inline indirect-branch lookup + RAS; Stage 4 = superblocks +
  reg cache + peephole. Each stage shippable and measured. This is the port
  playbook shape (though for Hercules the VCC ladder replaces the translator
  content).
- **`tools/dbt5/`** — the arch-neutral/target split (`pre/{lift,ssa,mir}` vs
  `target/{x64,a64}/{burg,lir,regalloc,codegen}`) is the intended extension
  slot if a real optimizing tier is ever wanted.
- **Verification culture:** `shadow_interp` paranoid/paranoid-lite lockstep
  modes; `regression/run-differential.sh` N-way engine differential harness;
  `INTEL_AGENT_HANDOFF.md` is a genuine DBT-debugging handbook (flag-bisection
  matrix, trace env vars, hardware-watchpoint recipes).
- **Measurement hygiene:** *"A BIPS figure without a machine, a build size,
  and a date is not a measurement — it's a rumor with units."*
  (`docs/EMULATORS.md`). Adopt verbatim.
- **Two register-allocator designs side by side** (x64 demand-driven LRU vs
  a64 static prescan) with the cross-project back-edge-corruption postmortem
  (`tools/dbt/ISSUES.md:76`) — read before designing back-edges.
- **Measured datum that matters here:** bounds checks + W^X cost only
  **2–8%** — evidence that an inlined guard/probe on the memory path is
  bearable.

**Numbers:** 7.50 BIPS (M5 Max) / 4.07 BIPS (Cascade Lake) on benchmark_core;
~2.3 host insns per guest insn; real workload (Ragel CSV) 326–351 MB/s vs 513
native. The sealed-universe bootstrap (`selfhost/`, stage08 cross-compilers
building the DBT inside the guest) is philosophically adjacent but not on the
Hercules critical path.

---

## ~/riscv — the cleanest skeleton, the least applicable invariants (RV32IMFD)

**Why it still matters:** the cleanest common/arch file split
(`dbt_common.c` / `dbt_x64.c` / `dbt_a64.c` / `shadow.c` / `interp.c`), and
the emitter pair (`emit_a64.h` 1102 lines / `emit_x64.h` 975 lines) that
proved portable across guests. The superblock **side-exit snapshot** mechanism
(defer the taken arm, snapshot register-cache state, emit cold stubs that
replay dirty writebacks) is the right *shape* for materializing precise state
at arbitrary exits.

**What it never had to confront — precisely the Hercules hard parts:** no
flags, no MMU (guest loads are literally one host instruction, unchecked),
no interrupts, no exceptions (untranslatable → terminate, "there is no
interpreter handoff path"), no SMC handling, bump-only code cache that
`exit(1)`s when full. Its design reads as a template for the emitter/dispatch/
cache layer, not for system emulation.

**Verifier:** `-V` lockstep shadow with a 4096-entry store buffer, forced
unchained dispatch, three-way semantic sharing of divergence-prone helpers
(fclass/fcvt) between interpreter, shadow, and both JITs. ~266 tests across 8
ported real programs; differential vs qemu-riscv32.

**Numbers (from its CLAUDE.md, which retracts its own stale claims):** 9.10–
9.3 BIPS benchmark_core on M5 Max (a64); **3.11 BIPS** on Cascade Lake — the
"6.2 BIPS x86-64" figure in `docs/linkedin-post.md` is retracted; realistic
workloads 1.0–4.6× vs qemu-user. Its own conclusion: *"the translator
comparison is a function of (host, workload); no constant exists."* Anchor
Hercules projections on the realistic-workload end, not the microbenchmark.
