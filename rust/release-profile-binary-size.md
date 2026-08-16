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
