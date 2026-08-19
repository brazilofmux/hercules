# DBT for Hercules — feasibility study

> **The project is now live.** Development happens in the hyperion fork
> at **https://github.com/brazilofmux/hyperion** (branch `dbt`, docs
> under `dbt/`), starting with the lockstep verifier
> (`dbt/DESIGN-VERIFIER.md`). This directory remains as the frozen
> feasibility study referenced by the 2026-08 hercules-390 list post;
> the data in `data/` is canonical in both places.

**Date:** 2026-08-19. Survey of SDL-Hercules-390/hyperion internals plus the four
in-house DBT projects (`~/riscv`, `~/z80`, `~/vcc`, `~/slow-32`), asking: can the
same dynamic-binary-translation techniques be applied to Hercules?

**Verdict: yes — but the donor architecture is VCC's, not riscv's.** The
headline BIPS numbers (riscv 9.3, z80 4.3) come from translators exploiting
invariants z/Architecture denies: fixed-format instructions, no MMU, no
interrupts, no flags, unchecked single-instruction memory access, never-written
code. What transfers instead is VCC's hybrid model — a JIT bolted additively
onto a mature interpreter with no flag day — because Hercules's ~1,300
`DEF_INST(BYTE* inst, REGS* regs)` handlers are ready-made trampoline targets,
plus z80's static condition-code dead-store elimination, plus slow-32's
scaffolding (block cache, verifier, differential harness, staged port plan).

Projected win (unmeasured — see [assessment.md](assessment.md) §7 for the
reasoning and the caveats): **3–8× over the interpreter on hot problem-state
code**, degrading toward 1× on I/O-bound and supervisor-heavy work. No JIT work
has ever been attempted in hyperion (nothing in `_TODO.txt` or `CHANGES`).

## Contents

| Doc | What it holds |
|---|---|
| [assessment.md](assessment.md) | The synthesis: proposed architecture, the load-bearing design decision (memory-resident guest state + longjmp precision), the three genuinely new problems, performance projection, build order |
| [hercules-internals.md](hercules-internals.md) | Recon of the hyperion execution core: dispatch loop, AIA, `maddr_l` software TLB, `EX`, program-check delivery, SMP model |
| [donors.md](donors.md) | Per-project inventory of the four DBT codebases: what transfers, what doesn't, measured numbers, key files |

## Provenance

- hyperion surveyed at commit `ae9b497` ("Release 4.9.1 in progress...",
  2025-12-07, master as fetched 2026-08-19). All `file:line` references in
  these docs are against that commit. The `sdl/Dockerfile` here builds from
  master at whatever HEAD is current, so expect drift.
- Donor-project findings are from source surveys run 2026-08-19; the z80
  benchmark numbers were re-run and confirmed on disk, the others are quoted
  from each repo's own docs (which are diligent about machine/date caveats —
  keep that discipline here).

## Repo status

This repo currently holds only Dockerfiles. These docs live here until the DBT
work gets its own ~-level project (a fork of SDL-Hercules-390/hyperion); they
are written to be lifted out wholesale at that point.
