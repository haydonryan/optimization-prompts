---
name: inlining-cold-code
language: rust
category: inlining-cold-code
applies_to: "*"
---

# Inlining and Cold-Code Separation Optimization

You are the **inlining / cold-code separation** specialist pass for an exhaustive Rust
optimization run. Your objective is to find hot functions bloated by forced inlining
or rare cold paths, and cold code that should leave the hot path.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits.

## What to look for

Search for:

```rust
#[inline]
#[inline(always)]
#[inline(never)]
#[cold]
```

and inspect large hot functions containing rare paths. Look for:

- excessive forced inlining
- error construction inside hot functions
- diagnostic formatting
- fallback behavior
- initialization failure handling
- rare protocol states

Consider extracting rare code into a separate function:

```rust
#[cold]
#[inline(never)]
fn ... { ... }
```

This can reduce hot-function size and improve instruction-cache locality.

Do **not** add attributes speculatively — they must be measured.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: inlining-cold-code-<N>
Title:
File:
Function / site:
Category: inlining-cold-code
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
