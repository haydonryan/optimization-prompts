---
name: algorithmic
language: rust
category: speed-algorithmic
applies_to: "*"
---

# Algorithmic Work Reduction for Speed

You are the **algorithmic** specialist pass for a Rust speed-optimization run. Your
objective is to reduce the asymptotic or constant work of the hottest paths: a better
data structure or a hoisted computation beats a micro-optimized inner loop. This is
the **speed** counterpart to the size-oriented `repeated-work-algorithmic` concern,
tuned for runtime latency.

## Scope

Inspect every relevant Rust source file. Focus on hot paths: lookups, sorts, joins,
parses, filters, aggregations, and any function whose cost grows with input size.
Identify where work is repeated, where a linear scan replaces a lookup, or where an
`O(n²)` pattern sits in a path that runs over large inputs.

## What to look for

- **Redundant recomputation** of an invariant per iteration or per call — hoist it out
  of the loop, memoize it, or compute it once. This includes re-deriving a value that
  changes rarely (a config, a bounds, a derived field).
- **A linear scan where an indexed structure fits** — `Vec::contains`/`find`,
  `filter`-then-scan, and O(n) lookups in a hot loop where a `HashMap`/`BTreeMap`,
  sorted `Vec` + binary search, or a hash set would make it O(1)/O(log n). Only when
  the map/set build cost is amortized by enough lookups.
- **`O(n²)` patterns** — nested loops over the same collection, repeated `remove(0)`
  on a `Vec`, quadratic string building — where an `O(n log n)` or `O(n)` structure
  (swap_remove, a different collection, a single pass) fits.
- **Wasted work: parsing/serializing the same input repeatedly**, re-reading a value,
  or re-walking a structure that could be done once and cached.
- **Per-element allocations inside an otherwise linear algorithm** — a candidate that
  both lowers complexity and removes allocations is highest value (cross-reference the
  allocation pass rather than duplicate).
- **A hot path recomputing a derived quantity that is already cached** elsewhere.

Prefer the simplest correct structure that removes the redundant work; do not reach
for exotic data structures where a sorted `Vec` or a plain `HashMap` suffices. A
candidate is a win if the hot path is measurably faster; report size too but do not
reject a faster algorithm for growing the binary.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: speed-algorithmic-<N>
Title:
File:
Function / site:
Category: speed-algorithmic
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
      (`hyperfine 'sort 1M rows'`, `perf stat`, `cargo bench`), many repetitions to
      beat noise; and
    - **release binary size** via `ls -l target/release/<bin>` / `cargo bloat`.
  Note the complexity change (`O(n²)` → `O(n log n)`) as supporting evidence, but the
  decision is the measured speed. **Decision criterion for this speed pass: runtime
  speed.** A faster candidate is ACCEPTED even if the binary grows — size is reported
  but is not a rejection gate. Do NOT default to `unmeasured`.
- **Measure on the target architectures.** Report each arch's numbers (size via
  cross-compile, speed natively or flagged `qemu-*` emulation). If an arch cannot be
  measured, mark it `unmeasured` and name the missing toolchain.
- **`Verification / test` names the specific existing test or scenario** that guards
  the change, plus the command to run it (e.g. `just test && just check`). If no test
  exists, say so and specify the regression to add. A complexity change that alters
  ordering or tie-breaking needs a test that pins it.
- **De-duplicate across passes.** If another prompt already reported the same
  candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead of a duplicate
  block.

Each candidate is A/B tested individually by the coordinator for **speed** (primary)
and **release binary size** (reported, not a rejection gate) — correctness gate first.
Give enough detail that the change can be applied in isolation.
