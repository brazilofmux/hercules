# Instruction-mix measurements — raw data

Runs performed 2026-08-19. Method summary (full narrative in
[../measurement-post-draft.md](../measurement-post-draft.md)):

- **Emulator:** SDL-Hercules-390/hyperion at commit `ae9b497`, built
  natively on macOS arm64 with Apple clang 21 at `-O3`, with one source
  change: `featall.h:78` flipped from `#undef` to
  `#define OPTION_INSTR_COUNT_AND_TIME`, enabling the in-tree `icount`
  per-opcode counters (exact counts; the option also timestamps each
  instruction, so timing columns are perturbed and indicative only).
- **Guest:** MVS 3.8j TK5 update 5, S/370 mode, `NUMCPU 1`,
  `MAINSIZE 16`M, IPL'd unattended via `scripts/ipl.rc`; jobs submitted
  through the sockdev reader on port 3505; icount controlled through the
  HTTP console (`/cgi-bin/tasks/syslog?command=...`).
- **Host:** Apple M5 Max MacBook Pro, 48 GB, macOS 26.5.2.
- **Protocol per workload:** `icount clear` + `icount enable` → submit
  job → wait for `IEF404I`/`$HASP250` on the console log →
  `icount disable` → `icount` (display) → capture the `HHC02292I` lines.
  Counts therefore cover the whole system (MVS + JES2 + job) during the
  window; the measured idle floor (~104K instructions/s, see
  `idle-icount.txt`) bounds background pollution below ~1% of either
  workload sample.

## Files

| File | Workload | Total instructions |
|---|---|---|
| `asmfc-icount.txt` | HERCASM: 3-step Assembler XF (IFOX00/ASMFC) job, 5,600 generated statements per step, all clean | 87,947,689 (116 distinct opcodes) |
| `cobol-icount.txt` | CPUMIX: COBOL (COBUCLG) compile-link-go; GO = 10M iterations of a packed-decimal COMP-3 paragraph | 379,852,458 (115 distinct opcodes) |
| `idle-icount.txt` | 60 s of idle IPL'd TK5 (no jobs) | 6,229,656 (97 distinct opcodes) |
| `batch-icount.txt` | BATCH: a real production month-end general-ledger run — 37 steps: 16 SORTs + 19 COBOL GL programs + JCL overhead (captured 2026-08-19 on the user's live TK5 system) | 95,172,887 (118 distinct opcodes) |

Line format is hyperion's own `icount` output, sorted descending:
`Inst 'hh' count N (p%) time T (avg) MNEMONIC template handler_name`.
Percentages in the raw lines are integer-truncated; recompute from the
counts for precision.

## Headline coverage numbers

- HERCASM: top 10 opcodes = 66.1%, top 20 = 82.1%, top 40 = 95.6%.
- CPUMIX: top 10 = 85.0%, top 20 = 99.2%, top 40 = 99.8%; the six
  decimal ops (ZAP, AP, MP, CP, MVO, DP) total 39.5%.
- BATCH (real production month-end run): top 10 = 67.4%, top 20 =
  80.6%, top 40 = 94.2% — and the **decimal family is only 0.2%**.
  The real commercial batch mix looks like the compiler mix (BC 21.7%,
  L 14.4%, TM/LR/ST/LA/SLR/BCR/LTR/LM), not like the synthetic
  decimal loop: SORT passes, record handling, and OS/JES overhead
  dwarf the per-record arithmetic. Implication for any JIT: the same
  ~40-opcode inline set covers all three workloads, and decimal
  handlers can stay interpreter/helper calls at no measurable cost.
  Caveat: this application deliberately avoids packed decimal by
  design (binary linear-day date math in assembler, per Calendrical
  Calculations, epoch 1601). Typical commercial COBOL is far more
  COMP-3-heavy; treat CPUMIX (39.5% decimal) and BATCH (0.2%) as
  brackets on the range, not either one as "typical."
- Rate for scale (counting off): session maxrates peak 82 MIPS; the
  CPUMIX job ≈ 42 MIPS effective over its 9 s wall clock including
  compile/link I/O. BATCH runs ~3 s wall instrumented or not — it is
  not CPU-bound at these volumes.
