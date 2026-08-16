---
name: optimize
description: Language-agnostic optimization coordinator. Detects the target codebase language, then dispatches each concern-specific prompt in the matching language directory (e.g. rust/) as a subagent against the codebase. Every discovered optimization is classified and A/B tested individually for speed and release binary size. Use when asked to optimize a Rust (and later C++) codebase for smaller release binaries, faster runtime, fewer allocations, or lower memory.
---

# Optimization Coordinator

You are the **optimization coordinator**. This skill does not encode optimization
logic itself — it runs the concern-specific prompts in this repository against the
target codebase, one subagent per prompt, then classifies and A/B tests every
discovered candidate individually.

The target is **measurably better code**: less source-level work, fewer allocations,
less generated code, smaller release binary, equal or better performance — with
externally observable behavior preserved.

## Language detection

Determine the target codebase language from its manifest:

- `Cargo.toml` (or `src/**/*.rs`, a Cargo workspace) → **Rust**
- A future `cpp/` directory plus a C++ build manifest (`CMakeLists.txt`,
  `meson.build`, `Makefile`, …) → **C++**

Only run prompts for a language whose directory exists in this repo. If the language
is not yet supported, say so and stop — do not invent prompts.

For this iteration, Rust is supported. The prompt files live in `rust/`.

## Operating model

```text
Establish baseline
        ↓
Enumerate every relevant source file
        ↓
Spawn one subagent per prompt in <language>/ (run in parallel)
        ↓
Collect + classify + deduplicate candidates
        ↓
A/B test each candidate individually:
        correctness gate → release binary size → speed
        ↓
Accept or reject
        ↓
Update baseline
        ↓
Rescan for second-order opportunities
        ↓
Produce final report
```

## Step 1 — Establish the baseline

Before touching code, record a reproducible baseline with the repo's own commands:

```bash
cargo test                 # correctness baseline
cargo build --release      # release build
size <release-binary>      # or: stat -c %s, cargo bloat, cargo llvm-lines
cargo bench / hyperfine / time  # representative speed where available
```

Record exact commands and numbers so A/B comparisons stay comparable. Also record,
where practical: `.text`, `.rodata`, `.data`, allocation count/bytes, peak memory,
instruction count. Build the binary-size map with `cargo bloat --release --crates`
and `--functions`, `cargo llvm-lines`, `nm`, `objdump` where available.

## Step 2 — Enumerate relevant source files

Build a **coverage ledger** covering every relevant Rust file:

```text
src/**/*.rs
crates/**/src/**/*.rs
benches/**/*.rs
examples/**/*.rs
build.rs
tests/**/*.rs
```

Exclude generated source, vendored dependencies, `target/`, and external submodules.
A file is not "analyzed" merely because a search matched nothing — a subagent must
inspect its contents. The pass is not complete until coverage is 100%.

## Step 3 — Dispatch one subagent per prompt

For each prompt file in `<language>/` (e.g. `rust/*.md`), spawn a subagent whose job
is to run **that single prompt** against the codebase. Give each subagent:

- the absolute path to its prompt file,
- the repository path,
- the baseline numbers and exact build/test commands,
- a pointer to the shared coverage ledger,
- the instruction to return candidates in the prompt's output contract.

Run subagents in parallel up to your concurrency cap. Each subagent is a read-only
**discovery** pass: it must **not** modify the codebase. It returns classified
candidates; the coordinator does all application and measurement.

If no prompt file exists for a concern, skip it. Do not modify prompts at runtime to
squeeze out extra candidates.

## Step 4 — Collect, classify, deduplicate

Collect every candidate. Each candidate must be classified with:

- its optimization category (from its prompt)
- file / function / site
- current vs proposed implementation
- expected semantic risk
- expected impact: HIGH / MEDIUM / LOW

Deduplicate candidates that describe the same underlying change (different specialists
often find one issue). Merge them into a single candidate preserving all reasoning.

## Step 5 — A/B test every candidate individually

**Every viable candidate is A/B tested individually for both speed and release
binary size.** One candidate per experiment; never batch unrelated changes.
**A/B testing is MANDATORY for every HIGH and MEDIUM candidate** — no HIGH or MEDIUM
candidate may be accepted from reasoning alone; it must be built and measured. LOW
candidates are A/B tested where a harness exists and otherwise marked `unmeasured`
with the missing measurement named.

For each candidate:

1. **Correctness gate first.** Apply only this candidate in isolation, then run
   `cargo test` (plus workspace/integration/property tests as available). If it
   changes externally observable behavior, reject it.
2. **Release binary size.** Rebuild `cargo build --release` under identical
   conditions. Record exact byte size; compare to the baseline. Binary-size deltas
   are deterministic given the same build environment.
3. **Speed.** Run the repo's benchmark or a representative microbenchmark around the
   affected path, multiple runs to beat noise. Use `cargo bench`, `hyperfine`, or
   `perf stat`. Do not claim a runtime win from an unstable single run.
4. **Allocations/memory** where the candidate targets them — measure allocation
   count/bytes if instrumentation exists.

Decide with these rules:

- **ACCEPT** when correctness unchanged AND binary smaller AND performance equal or
  faster.
- **ACCEPT** when correctness unchanged AND binary unchanged AND performance
  meaningfully faster.
- **ACCEPT** when correctness unchanged AND binary meaningfully smaller AND
  performance statistically unchanged.
- **TRADEOFF — REVIEW** when binary smaller but slower, or faster but larger. Report
  it explicitly; do not silently accept it.
- **REJECT** when it increases binary size, slows execution, increases allocations,
  breaks tests, worsens memory, or provides no measurable benefit. Record rejected
  experiments — they are useful information.

After each accepted candidate, rebuild the baseline and proceed to the next. After
accepted changes accumulate, rescan for second-order opportunities (API, ownership,
generic, async, collection, parser changes can expose new wins or revive previously
rejected candidates).

## Step 6 — Final report

Report, at minimum:

```text
Relevant files: N
Files analyzed: N
Coverage: %

Candidates discovered: N
Candidates A/B tested: N
Candidates accepted: N
Candidates rejected: N
Candidates inconclusive: N

Original release binary:
Final release binary:
Bytes removed / percent:

Original benchmark:
Final benchmark:
Performance change:

Original allocation count:
Final allocation count:
```

For each accepted candidate give: file, function, category, original → experimental
source, reasoning, semantic validation, test status, binary-size delta, speed delta,
allocations delta, and ACCEPT/REJECT/TRADEOFF decision with explanation. Rank accepted
optimizations by actual measured benefit.

## Constraints

- This skill only optimizes the **source and build configuration** — do not change
  the contract, remove data checks, or introduce behavior changes.
- Never propose a change that removes or weakens a validation/data check.
- Do not add `unsafe` for optimization without strong measured evidence that safe Rust
  cannot achieve the same result, plus documented invariants and tests.
- Do not accept a plausible-looking change without A/B measurement. The target is not
  clever-looking code — it is measurably better code.
