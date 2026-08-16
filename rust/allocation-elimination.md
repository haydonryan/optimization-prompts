---
name: allocation-elimination
language: rust
category: heap-allocation
applies_to: "*"
---

# Heap Allocation Elimination

You are the **heap-allocation elimination** specialist pass for an exhaustive Rust
optimization run. Your objective is to find places where heap allocations can be
removed, reduced, or reused without changing externally observable behavior.

## Scope

Inspect every relevant Rust source file (`src/**/*.rs`, `crates/**/src/**/*.rs`,
`benches/**/*.rs`, `examples/**/*.rs`, `tests/**/*.rs`, `build.rs`). Scan actual file
contents, not just search hits.

## What to look for

Look for every allocation site and API that returns newly allocated values:

```rust
String  Vec  Box  Rc  Arc
format!  to_string()  to_owned()
String::from()  Vec::from()
collect()  collect::<Vec<_>>()  collect::<String>()
boxed()
```

For each allocation ask: **can this allocation disappear?** Then consider:

- borrowing instead of owning
- moving instead of cloning
- caller-provided buffers
- reusable buffers
- stack storage, fixed arrays, slices, `&str`
- `Cow`, `ArrayVec`, `SmallVec`
- direct streaming/writing instead of building an intermediate value
- iterator pipelines without `collect`
- reusing `String`/`Vec` capacity, preallocating once
- avoiding allocation inside loops
- changing an internal API to avoid ownership

Pay special attention to allocations inside loops, parsers, hot paths, request
handlers, serialization, formatting, and async loops.

Do not assume a stack-backed container is automatically better — additional generic
code can grow the binary. Propose it only as a candidate to be measured.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: allocation-elimination-<N>
Title:
File:
Function / site:
Category: heap-allocation
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
