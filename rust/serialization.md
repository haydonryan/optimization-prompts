---
name: serialization
language: rust
category: serialization
applies_to: "*"
---

# Serialization and Deserialization Optimization

You are the **serialization / deserialization** specialist pass for an exhaustive Rust
optimization run. Your objective is to find serialization code that copies, allocates,
or generates more code than necessary.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits.

## What to look for

Inspect Serde and other serialization code for:

- intermediate `Value` trees
- deserialize → copy → convert chains
- unnecessary ownership
- generic serializer duplication
- repeated serialization
- large generated implementations

Look for borrowing and direct representations when safe.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: serialization-<N>
Title:
File:
Function / site:
Category: serialization
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
