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
Measured impact (per target arch: binary size / speed, before -> after):
Verification / test:
Semantic risk (what could break / what to verify):
```

Requirements:

- **Code is mandatory — the FULL section, not a fragment.** `Current code (before)`
  and `Proposed code (after)` MUST each contain the COMPLETE enclosing function,
  method, or block — the full signature plus every statement/expression — together
  with enough surrounding context (relevant types, imports, and call sites) that the
  coordinator can apply the change verbatim without opening any other file. No
  ellipses (`...`), no truncated bodies, no single-line paraphrases, no prose standing
  in for code. If the change spans a whole type, module, or Cargo/profile block, show
  the complete before and after of that unit. A candidate whose before/after omits any
  part of the affected code is INCOMPLETE — the coordinator MUST reject it rather than
  reconstruct it.
- **Measurement is mandatory for BOTH binary size AND runtime performance — never
  speculative.** Do not write "may improve speed" or "may reduce size". Every candidate
  carries real before/after numbers for BOTH:
    - **release binary size** via `ls -l target/release/<bin>` / `cargo bloat` /
      `cargo llvm-lines`; and
    - **runtime performance** via the repo's benchmark harness if one exists, otherwise
      a concrete named microbenchmark or representative workload you specify with the
      exact command (e.g. `hyperfine 'sort large.txt'`, `perf stat`, `time`), run enough
      repetitions to beat noise.
  Record both in `Measured impact`. Do NOT default to `unmeasured`: every candidate must
  have a named performance measurement and a real before/after number. If a candidate
  genuinely has no measurable runtime path (e.g. a pure build/profile-flag change), say
  so explicitly and give the size result with that note. If the coordinator cannot
  measure in this environment, it marks the candidate `unmeasured` and names exactly
  which measurement is missing — but that is the exception, not the default.
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
