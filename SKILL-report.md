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

…

**Balance across optimization dimensions.** Optimization spans binary size, runtime
speed, codegen, monomorphization, data layout, control flow, algorithmics, AND heap
allocations. Heap-allocation findings are easy to over-report; do not let them crowd
out the other dimensions. When the hot path is already allocation-free, say so and do
not manufacture low-value allocation candidates. Prefer candidates that improve
multiple dimensions at once.

## Step 5 — A/B test each candidate

Collect every candidate. Classify each with category, file / function / site,
current vs proposed implementation, semantic risk, and expected impact. **Keep every
candidate, including LOW.** Deduplicate candidates that describe the same change,
merging reasoning.

**Full code is a hard gate.** Every candidate MUST arrive with COMPLETE
`Current code (before)` and `Proposed code (after)` — the full enclosing function,
method, or block (full signature + every statement) plus enough surrounding context
(types, imports, call sites) to apply verbatim without opening another file. No
ellipses, no truncated bodies, no single-line paraphrases, no prose standing in for
code. When a subagent returns a summary-only candidate or a fragment, the coordinator
MUST send it back to be completed — it must NOT reconstruct, infer, or accept it.

## Step 5 — A/B test each candidate

**A/B testing is MANDATORY for every candidate — HIGH, MEDIUM, and LOW — and it must
cover BOTH binary size AND runtime performance.** No candidate may be reported as
accepted (or as a claimed win) from reasoning alone. For each candidate, apply it in
isolation, run the correctness gate (`cargo test` / `just test && just check`), then
record real before/after numbers for BOTH:

- **release binary size** — `ls -l target/release/<bin>` / `cargo bloat` /
  `cargo llvm-lines`, per target architecture; and
- **runtime performance** — the repo's own benchmark harness if one exists; otherwise
  build or run a concrete named microbenchmark / representative workload with the
  exact command (e.g. `hyperfine 'sort large.txt'`, `perf stat`, `time`), multiple
  runs to beat noise.

If the repo has no performance harness for the affected path, the coordinator MUST
provision one (a benchmark, a representative workload, or a runnable script) rather
than leaving performance unmeasured. `unmeasured` is the exception, not the default:
**Exception — build-configuration candidates are NOT A/B-benchmarked.** Candidates
whose category is `codegen-flags` or `release-profile-binary-size` (LTO, `-C` flags,
`RUSTFLAGS`, `[profile.*]`, strip, panic strategy) are deterministic and low-risk:
binary size is fixed for a given build and they are not source transformations. Apply
each such suggestion in isolation, confirm the build and the repo's test gate pass,
and record the resulting binary size. Do **not** run the speed-measurement ceremony
on them; note any plausible runtime effect qualitatively.

## Step 6 — Produce the complete detailed report

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
- **Every candidate — HIGH, MEDIUM, and LOW — carries the FULL contract**, including
  both `Current code (before)` and `Proposed code (after)` (complete sections, never
  fragments) and a real `Measured impact` with actual before/after numbers for BOTH
  binary size and runtime performance per arch. No priority may be reduced to a
  summary-only bullet list or a bare title/impact line; a LOW candidate still gets
  its existing code, its proposed change, and its measured numbers, exactly like HIGH
  and MEDIUM. Compactness applies only to prose length, never to omitting the code or
  the measurements.
- Every candidate's status: `accepted` / `rejected` / `tradeoff` / `unmeasured` (the
  last only when measurement is genuinely impossible, with exactly which size and/or
  speed measurement is missing and on which arch named).
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
  a LOW candidate just because it is small, and do not report a LOW candidate without
  its full `Current code (before)` + `Proposed code (after)` contract.
