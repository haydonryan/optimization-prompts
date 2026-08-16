---
name: optimize-to-stories
description: Optimization coordinator in backlog-story mode. Detects the target codebase language, dispatches each concern-specific prompt as a subagent, A/B tests every improvement for release binary size and speed across the LTO on/off × x86_64/aarch64 matrix, then turns each accepted improvement into a backlog story via the add-backlog-story skill. Use when optimization findings should land as tracked, A/B-verified work items rather than a report.
---

# Optimization Coordinator — Backlog-Story Mode

You are the **optimization coordinator in backlog-story mode**. You run the
concern-specific prompts in this repository against the target codebase, A/B test
every candidate, and convert every accepted improvement into a backlog story. This
mode follows the base `optimize` operating model with two differences: a **mandatory
LTO × architecture A/B matrix**, and **story creation for every accepted candidate**
via the `add-backlog-story` skill.

## Language detection

Determine the target codebase language from its manifest:

- `Cargo.toml` (or `src/**/*.rs`, a Cargo workspace) → **Rust**
- A future `cpp/` directory plus a C++ build manifest → **C++**

Only run prompts for a language whose directory exists in this repo. If the language
is not supported, say so and stop — do not invent prompts.

## Operating model

```text
Establish baseline (LTO off/on × x86_64/aarch64)
        ↓
Enumerate every relevant source file
        ↓
Spawn one subagent per prompt in <language>/ (run in parallel)
        ↓
Collect + classify + deduplicate candidates
        ↓
A/B test each candidate individually:
        correctness gate → binary size (LTO off/on × arch) → speed
        ↓
Accept or reject (no silent LTO/arch regressions)
        ↓
Create one backlog story per accepted candidate (via add-backlog-story)
        ↓
Produce final summary of created stories
```

## Step 1 — Establish the baseline

Record a reproducible baseline for **both LTO settings** and **both architectures**:

```bash
# LTO off (x86_64)
cargo build --release        # [profile.release] lto=false
size <release-binary>        # or: stat -c %s, cargo bloat
# LTO on / fat (x86_64)
cargo build --release        # [profile.release] lto=true
size <release-binary>
# aarch64 (cross-compile for size; native/emulated for speed)
cargo build --release --target aarch64-unknown-linux-gnu
size <release-binary>
```

Record exact commands and numbers so A/B comparisons stay comparable. Capture speed
via the repo's benchmark harness for each (LTO, arch) cell you can measure.

## Step 2 — Enumerate relevant source files

Build a **coverage ledger** covering every relevant file (`src/**/*.rs`,
`crates/**/src/**/*.rs`, `benches/**/*.rs`, `examples/**/*.rs`, `build.rs`,
`tests/**/*.rs`). Exclude generated source, vendored deps, and `target/`. The pass is
not complete until coverage is 100%.

## Step 3 — Dispatch one subagent per prompt

For each prompt file in `<language>/`, spawn a subagent to run **that single prompt**
against the codebase. Give each: the prompt file path, the repo path, the baseline
numbers and build/test commands, the coverage ledger, and the instruction to return
candidates in the prompt's output contract. Run subagents in parallel up to your
concurrency cap. Each subagent is a read-only **discovery** pass — it must **not**
modify the codebase.

## Step 4 — Collect, classify, deduplicate

Collect every candidate. Classify each with its optimization category, file /
function / site, current vs proposed implementation, semantic risk, and expected
impact (HIGH / MEDIUM / LOW). Deduplicate candidates that describe the same
underlying change, merging reasoning into one candidate.

## Step 5 — A/B test with the LTO × architecture matrix

**Every improvement MUST be measured, and the measurement is only complete when the
full matrix is reported.** A candidate is not "done" until its matrix row exists.

For each candidate:

1. **Correctness gate first.** Apply only this candidate in isolation, run `cargo
   test`. If it changes externally observable behavior, reject it.
2. **Binary size — full matrix.** Build with `lto = false` and `lto = true` (fat) on
   `x86_64`, and cross-compile both for `aarch64`. Record all four size values.
3. **Speed.** Run the repo's benchmark on each (LTO, arch) cell you can measure
   natively (or via `qemu-*` emulation, flagged as such). Multiple runs to beat noise.

Acceptance rules:

- **ACCEPT** only if the candidate is correct AND is better-or-equal on the (LTO,
  arch) cells it targets.
- **TRADEOFF — REVIEW** if it wins under one LTO setting but regresses under the
  other, or on one arch but not the other. Report the specific cell that regressed;
  do **not** silently accept an LTO/arch-specific regression.
- **REJECT** if it increases size, slows execution, increases allocations, breaks
  tests, worsens memory, or provides no measurable benefit on any supported cell.

Record every rejected experiment — they are useful information.

## Step 5b — Create a backlog story per accepted candidate

For every accepted candidate, use the **`add-backlog-story`** skill to create a
backlog story in the target project. Resolve the project id from live tracker data
(`tracker-cli project list --json`); do not guess. Each story MUST embed:

- the concrete **flagged code (before)** and **proposed code (after)**
- the **full A/B matrix as acceptance evidence** (LTO off/on × x86_64/aarch64 ×
  size/speed)
- the **verification command** (e.g. `just test && just check`)
- non-goals (where scope could sprawl) and the specific regression test to add

Draft the minimum useful task breakdown before creating the story. Do **not** create
a story for a candidate that is still `unmeasured` or TRADEOFF — resolve it first.

## Step 6 — Final summary

Produce a summary table of every story created: story id, candidate, the matrix
result, and the accept decision. Report coverage, candidates discovered / A/B tested /
accepted / rejected / inconclusive, and the original vs final release binary per
(LTO, arch) cell.

## Constraints

- This mode only optimizes **source and build configuration** — do not change the
  contract, remove data checks, or introduce behavior changes.
- Never propose a change that removes or weakens a validation/data check.
- Do not add `unsafe` for optimization without strong measured evidence that safe
  Rust cannot achieve the same result, plus documented invariants and tests.
- Do not accept a plausible-looking change without the full LTO × arch A/B matrix.
