---
name: compact-idiomatic-code
language: rust
category: compact-idiomatic-code
applies_to: "*"
---

# Compact Idiomatic Code

You are the **compact idiomatic code** specialist pass for an exhaustive Rust
optimization run. Your objective is to find small, local code that can be made more
compact and idiomatic, reducing branches, lookups, allocations, and generated code
without reducing clarity or removing data checks.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits.

## What to look for

- Delete dead locals, unused helper functions, redundant branches, unreachable match
  arms, duplicate formatting, and temporary variables that only obscure ownership.
  **Preserve data checks.**
- Replace clone-heavy APIs with borrowed parameters, `as_ref`, `as_deref`, `Cow`,
  slices, or references when lifetimes stay simple.
- Avoid `collect` when an iterator can be consumed directly; avoid intermediate `Vec`,
  `String`, or `HashMap` values in hot paths.
- Prefer `Entry` APIs, `retain`, `extend`, `drain`, `mem::take`, `Option::take`,
  `is_some_and`, `then_some`, `map_or`, `ok_or_else`, and slice methods when they
  reduce branches, lookups, or allocations without reducing clarity.
- Prefer `sort_unstable`, `binary_search`, preallocation, `with_capacity`, `reserve`,
  or stack arrays where ordering, size, and workload make them correct.
- Avoid repeated `format!`, `to_string`, `to_owned`, `String::from`, regex
  compilation, path normalization, env reads, and serialization in loops.
- Move expensive error/log message construction behind cold branches with closures
  such as `ok_or_else` or `with_context` where applicable.
- Replace hand-written loops with iterator adapters only when the generated work is
  equal or lower and the result is clearer. Keep loops when they avoid allocation or
  short-circuit more directly.
- Check `unwrap`/`expect` in runtime paths. Do **not** remove the check; convert it to
  a typed error or explicit validation path when user input, files, network, or
  config can trigger it.

Never remove or weaken a data check (validation, parsing, bounds, auth/permission,
integrity, invariant, error handling, timeout, defensive checks). If a check looks
duplicated, treat it as a correctness boundary until proven otherwise; at most
propose a verified consolidation that preserves the same checks.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: compact-idiomatic-code-<N>
Title:
File:
Function / site:
Category: compact-idiomatic-code
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
