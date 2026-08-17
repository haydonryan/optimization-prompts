---
name: release-profile-binary-size
language: rust
category: release-profile-binary-size
applies_to: "*"
---

# Release Profile and Build-Config Binary Size

You are the **release profile / build config** specialist pass for an exhaustive Rust
optimization run. Your objective is to find build-configuration changes that shrink
the release binary, distinct from source-level optimization.

## Scope

Read `Cargo.toml` release profiles, workspace profiles, `.cargo/config.toml`, and
per-crate feature settings.

## What to look for

Inspect release profile settings:

```toml
[profile.release]
lto
codegen-units
panic = "abort"
strip
debug / debug-assertions
opt-level
```

and per-crate settings. Use `cargo tree`, `cargo bloat --release --crates` /
`--functions`, and feature analysis to identify heavy dependencies and symbols.

Look for:

- `lto` (thin/fat) and `codegen-units = 1` where they reduce size without hurting
  runtime meaningfully
- `panic = "abort"` where panic=unwind is not required
- `strip = true` / `strip = "symbols"` for release artifacts
- `opt-level = "z"` or `"s"` where size is the priority
- disabling debug symbols and `debug-assertions` in release
- disabling unused default features and gating optional integrations

Do **not** trade away meaningful runtime performance for binary size unless the user
asked for that tradeoff or measurements make it worthwhile. This pass covers build
configuration only — source-level optimization is handled by the other specialist
passes.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: release-profile-binary-size-<N>
Title:
File:
Function / site:
Category: release-profile-binary-size
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
