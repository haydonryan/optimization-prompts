---
name: monomorphization
language: rust
category: monomorphization
applies_to: "*"
---

# Generic and Monomorphization Optimization

You are the **generic / monomorphization** specialist pass for an exhaustive Rust
optimization run. Your objective is to reduce duplicated machine code caused by
generic functions being instantiated for many concrete types.

## Scope

Inspect every relevant Rust source file. Use `cargo llvm-lines` and `cargo bloat`
where available to find heavily instantiated generic functions.

## What to look for

Search for:

- large generic functions
- heavily reused generic helpers
- generic parsing / serialization / iterator / error-handling / async functions
- generic functions called with many concrete types

Determine whether a generic function causes substantial duplicated machine code.
Consider separating a **small generic adapter** from a **large non-generic
implementation** so the body is not recompiled for every `T`. Consider coarse-grained
dynamic dispatch only where appropriate.

Do **not** mechanically replace generics with `dyn Trait`. Trait objects can hurt
performance and grow call overhead. Only propose a change that is likely to reduce
binary size without a performance regression.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: monomorphization-<N>
Title:
File:
Function / site:
Category: monomorphization
Priority: HIGH | MEDIUM | LOW
Description:
Current code (before):
Proposed code (after):
Measured impact (per target arch: binary size / speed, before -> after):
Verification / test:
Semantic risk (what could break / what to verify):
```

Requirements:

- **Code is mandatory.** `Current code (before)` and `Proposed code (after)` MUST be
  concrete, apply-able code snippets (Rust unless stated otherwise) — not prose
  summaries. If a candidate genuinely cannot be expressed as code, say so explicitly
  and show the surrounding call site.
- **Measurement is mandatory, not speculative.** Do not write "may improve speed" or
  "may reduce size". Apply the change in isolation and measure: release binary size
  via `cargo bloat` / `ls -l target/release/<bin>` (or `cargo llvm-lines` where
  relevant), and speed via the repo's own benchmark/profiling harness if one exists.
  Record real before/after numbers in `Measured impact`. If you cannot measure in
  this environment, mark the candidate `unmeasured` and name exactly which
  measurement is missing.
- **Measure on the target architectures.** A/B results are architecture-specific:
  layout, alignment, `target-cpu`, and vectorization differ between `x86_64` and
  `aarch64`. Build and measure on every arch the software ships on (commonly both).
  Binary size is measurable for any arch via cross-compilation
  (`cargo build --target <triple>`); runtime speed requires native execution (or
  `qemu-*` emulation, flagged as such). Report the arch each number came from; if an
  arch cannot be measured, mark it `unmeasured` and name the missing toolchain.
- **`Verification / test` names the specific existing test or scenario** that guards
  the change, plus the command to run it (e.g. `just test && just check`). If no
  test exists, say so and specify the regression to add.
- **De-duplicate across passes.** If another prompt already reported the same
  candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead of a duplicate
  block.

Each candidate is A/B tested individually by the coordinator for **speed** and
**release binary size** (correctness gate first). Give enough detail that the change
can be applied in isolation.
