---
name: optimize-to-report
description: Optimization coordinator in detailed-report mode. Detects the target codebase language, dispatches each concern-specific prompt as a subagent, A/B tests candidates for release binary size and speed (per architecture), and produces a complete written report of ALL candidates — HIGH, MEDIUM, and LOW — with flagged code, proposed remediation, per-arch A/B evidence, and verification. Use when you want a thorough report file, not tracker stories.
---

# Optimization Coordinator — Detailed-Report Mode

You are the **optimization coordinator in detailed-report mode**. You run the
concern-specific prompts in this repository against the target codebase, A/B test
candidates, and write a **complete detailed report**. This mode differs from the base
`optimize` skill in one essential way: **the report includes every candidate,
including LOW-priority ones. Do not drop low issues.**

## Language detection

Determine the target codebase language from its manifest:

- `Cargo.toml` (or `src/**/*.rs`, a Cargo workspace) → **Rust**
- A future `cpp/` directory plus a C++ build manifest → **C++**

Only run prompts for a language whose directory exists in this repo. If the language
is not supported, say so and stop — do not invent prompts.

## Operating model

```text
Establish baseline (release binary size + speed, per architecture)
        ↓
Enumerate every relevant source file
        ↓
Spawn one subagent per prompt in <language>/ (run in parallel)
        ↓
Collect + classify + deduplicate candidates (keep ALL priorities)
        ↓
A/B test each candidate individually:
        correctness gate → release binary size → speed
        ↓
Produce a complete detailed report (HIGH + MEDIUM + LOW)
```

## Step 1 — Establish the baseline

Record a reproducible baseline with the repo's own commands (`cargo test`, `cargo
build --release`, size/bloat, the benchmark harness). Record binary size and speed
**per target architecture** (commonly `x86_64` and `aarch64`): size via
cross-compilation, speed natively or via flagged `qemu-*` emulation.

## Step 2 — Enumerate relevant source files

Build a **coverage ledger** covering every relevant file. Exclude generated source,
vendored deps, and `target/`. The pass is not complete until coverage is 100%.

## Step 3 — Dispatch one subagent per prompt

For each prompt file in `<language>/`, spawn a subagent to run **that single prompt**
against the codebase, returning candidates in the prompt's output contract. Run
subagents in parallel up to your concurrency cap. Each subagent is read-only.

## Step 4 — Collect, classify, deduplicate

Collect every candidate. Classify each with category, file / function / site,
current vs proposed implementation, semantic risk, and expected impact. **Keep every
candidate, including LOW.** Deduplicate candidates that describe the same change,
merging reasoning.

## Step 5 — A/B test each candidate

A/B test every candidate individually: correctness gate → release binary size →
speed, per target architecture. Do not claim a win from reasoning alone (see the
A/B-only gates in the empirical prompts). Mark unmeasured candidates `unmeasured`
with the missing measurement named; never silently drop a LOW candidate for being
small.

## Step 6 — Produce the complete detailed report

Write the report to a file (e.g. `<project>-opt.md`). It MUST include **ALL
candidates — HIGH, MEDIUM, AND LOW**. Do not drop low issues; rank them and note
expected impact, but keep every one.

Each candidate carries the full contract:

```text
Candidate ID:
Title:
Priority: HIGH | MEDIUM | LOW
File / function / site:
Description:
Current code (before):
Proposed code (after):
Measured impact (per target arch: binary size / speed, before -> after):
Verification / test:
Semantic risk:
```

Report structure:

- **Executive summary** of the highest-signal findings and a recommended execution
  order.
- **HIGH** candidates (full detail), **MEDIUM** (full detail), **LOW** (carry code +
  remediation + impact, but may be compact).
- Every candidate's status: `accepted` / `rejected` / `unmeasured` (with the missing
  measurement named).
- A coverage + A/B summary: candidates discovered / A/B tested / accepted / rejected /
  inconclusive, and original vs final release binary per arch.

The only output is the report — do **not** create tracker stories.

## Constraints

- This mode only optimizes **source and build configuration** — do not change the
  contract, remove data checks, or introduce behavior changes.
- Never propose a change that removes or weakens a validation/data check.
- Do not add `unsafe` for optimization without strong measured evidence that safe
  Rust cannot achieve the same result, plus documented invariants and tests.
- Do not accept a plausible-looking change without A/B measurement — and do not drop
  a LOW candidate just because it is small.
