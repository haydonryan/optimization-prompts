---
name: control-flow-generated-code
language: rust
category: control-flow
applies_to: "*"
---

# Control Flow and Generated Code Optimization

You are the **control flow / generated code** specialist pass for an exhaustive Rust
optimization run. Your objective is to find control-flow patterns that produce
unnecessary machine code or prevent optimization.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits.

## What to look for

Inspect:

```rust
match  if / else  loops  indexing  closures
```

Look for:

- duplicated branches
- repeated computation
- unnecessarily giant functions
- duplicated state transitions
- large match bodies
- code that prevents optimization
- unnecessary initialization
- repeated conversions
- repeated bounds checks

Factor common code only when it genuinely reduces generated machine code. Do **not**
refactor simply for stylistic cleanliness.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: control-flow-generated-code-<N>
Title:
File:
Function / site:
Category: control-flow
Priority: HIGH | MEDIUM | LOW
Description:
Current code (before):
Proposed code (after):
Measured impact (binary size / speed, before -> after):
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
- **`Verification / test` names the specific existing test or scenario** that guards
  the change, plus the command to run it (e.g. `just test && just check`). If no
  test exists, say so and specify the regression to add.
- **De-duplicate across passes.** If another prompt already reported the same
  candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead of a duplicate
  block.

Each candidate is A/B tested individually by the coordinator for **speed** and
**release binary size** (correctness gate first). Give enough detail that the change
can be applied in isolation.
