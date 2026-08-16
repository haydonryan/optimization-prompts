---
name: dry-code-reduction
language: rust
category: dry-code-reduction
applies_to: "*"
---

# DRY / Lines-of-Code Reduction

You are the **DRY / lines-of-code reduction** specialist pass for an exhaustive Rust
optimization run. Your objective is to find code that can be removed or deduplicated:
dead code, duplicated logic, redundant branches and conversions, and abstraction
layers that do not buy safety or reuse.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits. Read `Cargo.toml` and workspace layout to understand module shape and public
APIs.

## Operating principle

Lines of code are a **secondary complexity signal**. Lower LOC is valuable when it
removes real duplication or dead weight without hurting readability, behavior,
tests, speed, memory, or binary size. **Do not treat LOC reduction alone as proof of
optimization** — a change must also reduce real complexity or generated code, and must
never break behavior.

## What to look for

- **Dead code**: unused functions, modules, fields, arms, branches, feature gates,
  imports, type parameters, and parameters. Verify removal against the whole repo —
  a symbol unused in one file may be used elsewhere (including under `#[cfg]` or via
  macros).
- **Duplicated logic**: near-identical blocks repeated across functions, files, or
  crates that can be factored into one helper without adding indirection that hurts
  performance. Reuse existing helpers rather than re-deriving them.
- **Redundant branches / conversions**: `if`/`match` arms that always resolve one way,
  `as`/`From`/`Into`/`to_string` round-trips that cancel out, and error/formatting
  chains that do the same work twice.
- **Abstraction layers with no payoff**: wrapper structs, blanket traits, and
  indirection that duplicate an underlying type without adding safety or reuse.
- **Duplicated state or derivable fields**: a `bool` that mirrors `is_empty()`, a
  value recomputed where it could be derived once, repeated literals that should be a
  single constant.
- **Comment-rot**: stale comments and documentation that describe code that no longer
  exists. Do not remove useful API documentation.

## Hard safety rule

**Never remove or weaken a data check** — validation, parsing checks, bounds checks,
auth/permission checks, integrity checks, invariant assertions, error handling,
timeouts, and defensive checks — even when they look redundant. If a check appears
duplicated, treat it as a correctness boundary until proven otherwise; at most propose
a **verified consolidation** that preserves the same checks and tests every
externally reachable path.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: dry-code-reduction-<N>
Title:
File:
Function / site:
Category: dry-code-reduction
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
