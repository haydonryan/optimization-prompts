---
name: codegen-flags
language: rust
category: codegen-flags
applies_to: "*"
---

# Codegen and Linker Flag Optimization

You are the **codegen / linker flags** specialist pass for an exhaustive Rust
optimization run. Your objective is to find `RUSTFLAGS`/`Cargo.toml` compiler and
linker settings that produce a smaller or faster release binary, beyond the manifest
`[profile.release]` knobs.

## Scope

Inspect `Cargo.toml` profile sections, `.cargo/config.toml`, build scripts, CI/build
pipelines, and any `RUSTFLAGS`/`LDFLAGS`/`-C` flags. Check whether the binary is
distributed to other machines or pinned to one machine (this determines portability
constraints).

## What to look for

Look for:

- `-C target-cpu=native` (enables CPU-specific instruction sets, e.g. SIMD) — only
  when the binary is machine-pinned or a runtime-dispatched baseline is used; flag
  the portability risk for distributed binaries
- `-C target-feature` additions/removals, `-C tune-cpu`
- LTO type: `lto = "fat"` vs `"thin"` (fat LTO is stronger for size, slower to
  compile; thin is a middle ground)
- `codegen-units`, `opt-level` interplay, `opt-level = "s"`/`"z"` (size-optimized) vs
  `3`, and `panic = "abort"` (removes unwinding machinery — flag that any panic now
  aborts the process)
- `strip` level: `"symbols"` vs `"debuginfo"` vs `"none"`, and `-C strip`
- linker settings: `-C link-arg=-Wl,--gc-sections` + `-C link-dead-code`, `-Z
  build-std` (custom libstd), `-C link-arg` for `-Wl,-z,relro`/`--as-needed`, or a
  faster/smaller linker (e.g. `lld`/`mold`) where available
- `overflow-checks`, `debug-assertions`, `incremental`, `split-debuginfo` in release
- `[profile.release.package.*]` overrides, and per-crate `panic`/`opt-level`
- duplicate dependencies or feature bloat surfaced here (cross-reference the
  dependency-feature-reduction pass rather than duplicating)

Do **not** trade meaningful runtime performance for size, or size for portability,
without flagging the tradeoff explicitly. Every flag change must be measured on the
real release binary and checked against the repo's `just test`/`just check` gate.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: codegen-flags-<N>
Title:
File:
Function / site:
Category: codegen-flags
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
  concrete, apply-able snippets (TOML/`RUSTFLAGS`/linker flags, or the manifest edit)
  — not prose summaries. If a candidate genuinely cannot be expressed as an edit, say
  so explicitly.
- **Measurement is mandatory, not speculative.** Do not write "may improve speed" or
  "may reduce size". Build the release binary and measure: size via `ls -l
  target/release/<bin>` / `cargo bloat`, and speed via the repo's own
  benchmark/profiling harness if one exists. Record real before/after numbers in
  `Measured impact`. If you cannot measure in this environment, mark the candidate
  `unmeasured` and name exactly which measurement is missing.
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
