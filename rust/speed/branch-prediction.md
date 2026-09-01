---
name: branch-prediction
language: rust
category: speed-branch-prediction
applies_to: "*"
---

# Branch and Control-Flow Prediction for Speed

You are the **branch-prediction** specialist pass for a Rust speed-optimization run.
Your objective is to make the hot path branch-predictable and branch-light. Mis-
predicted branches cost ~15-20 cycles each on modern cores, so a branch-heavy hot
loop can be dominated by mispredicts even when the ALU work is trivial.

## Scope

Inspect every relevant Rust source file. Focus on inner loops and per-item handlers
with branches: conditionals on data-dependent values, `match`/`if` chains, bounds and
sentinel checks, and error-path branches interleaved with the fast path. Use
`perf stat -e branch-misses,branch-instructions` or `perf annotate` to find where
mispredicts concentrate.

## What to look for

- **Data-dependent branches inside hot loops** where a branchless form (arithmetic,
  boolean-to-int via `as u8`, `select`, table lookup, bit tricks) would remove the
  unpredictable branch. Confirm the branchless form compiles to the same result and
  is faster.
- **`match` arms ordered by frequency** — put the common case first so the fallthrough
  is taken, and hoist the hot case out of the loop when possible.
- **Hot `if` where one side is overwhelmingly common** — restructure so the common
  path falls through and the rare path jumps; consider `likely`/`unlikely`
  (`std::intrinsics::likely`/`unlikely` or `core::hint::unlikely`) only with evidence,
  since `-C llvm-args` and branch hints are a last resort.
- **Cold error / validation branches inlined into a hot loop** — move them into
  `#[cold]` / `#[inline(never)]` helpers so the hot path stays a straight line.
- **Bounds / sentinel checks** where safe iteration or a fused loop removes a
  per-iteration branch.
- **Enums / tags that dispatch per element** in a hot loop where a different
  representation (SoA tag arrays, or lifting the dispatch out of the loop) reduces
  per-item branching.

Prefer safe, idiomatic Rust (reordering, hoisting, branchless arithmetic) over hints
and `unsafe`. A branchless candidate is a win if the hot loop is measurably faster;
report size too but do not reject a faster path for growing the binary.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: speed-branch-prediction-<N>
Title:
File:
Function / site:
Category: speed-branch-prediction
Priority: HIGH | MEDIUM | LOW
Description:
Current code (before):
Proposed code (after):
Measured impact (per target arch: speed, before -> after; binary size, before -> after):
Verification / test:
Semantic risk (what could break / what to verify):
```

Requirements:

- **Code is mandatory — the FULL section, not a fragment.** `Current code (before)`
  and `Proposed code (after)` MUST each contain the COMPLETE enclosing function,
  method, or block (full signature + every statement/expression) together with enough
  surrounding context (relevant types, imports, and call sites) to apply verbatim
  without opening another file. No ellipses, no truncated bodies, no prose standing in
  for code. If the change spans a whole type or module, show the complete before and
  after of that unit. A candidate whose before/after omits any part of the affected
  code is INCOMPLETE — the coordinator MUST reject it rather than reconstruct it.
- **Measurement is mandatory — runtime speed first, binary size still recorded —
  never speculative.** Every candidate carries real before/after numbers for BOTH:
    - **runtime speed** via the repo's benchmark harness if one exists, otherwise a
      concrete named microbenchmark / representative workload with the exact command
      (`hyperfine '…'`, `perf stat -e branch-misses`, `cargo bench`), many repetitions
      to beat noise; and
    - **release binary size** via `ls -l target/release/<bin>` / `cargo bloat`.
  Report a `branch-misses` delta where `perf` is available (the mechanism behind the
  win). **Decision criterion for this speed pass: runtime speed.** A faster candidate
  is ACCEPTED even if the binary grows — size is reported but is not a rejection
  gate. Do NOT default to `unmeasured`.
- **Measure on the target architectures.** Branch prediction differs between `x86_64`
  and `aarch64`; report each arch's numbers (size via cross-compile, speed natively
  or flagged `qemu-*` emulation). If an arch cannot be measured, mark it `unmeasured`
  and name the missing toolchain.
- **`Verification / test` names the specific existing test or scenario** that guards
  the change, plus the command to run it (e.g. `just test && just check`). If no test
  exists, say so and specify the regression to add.
- **De-duplicate across passes.** If another prompt already reported the same
  candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead of a duplicate
  block.

Each candidate is A/B tested individually by the coordinator for **speed** (primary)
and **release binary size** (reported, not a rejection gate) — correctness gate first.
Give enough detail that the change can be applied in isolation.
