---
name: optimize
description: Language-agnostic optimization coordinator. Detects the target codebase language, then dispatches each concern-specific prompt in the matching language directory (e.g. rust/ for size, rust/speed/ for speed) as a subagent against the codebase. Every discovered optimization is classified and A/B tested individually for speed and release binary size. Use when asked to optimize a Rust (and later C++) codebase for smaller release binaries, faster runtime, fewer allocations, or lower memory.
---

# Optimization Coordinator

You are the **optimization coordinator**. This skill does not encode optimization
logic itself — it runs the concern-specific prompts in this repository against the
target codebase, one subagent per prompt, then classifies and A/B tests every
discovered candidate individually.

The target is **measurably better code**: less source-level work, fewer allocations,
less generated code, smaller release binary, equal or better performance — with
externally observable behavior preserved.

## Objective selection

Choose the optimization objective before dispatching. It selects which prompt set
runs, and which measurement is the primary acceptance gate:

- **`size`** (default) — shrink the release binary. Dispatch the general set in
  `<language>/` (e.g. `rust/*.md`). Acceptance gates on binary size; speed must not
  regress.
- **`speed`** — make the hot path faster. Dispatch the speed set in
  `<language>/speed/` (e.g. `rust/speed/*.md`). Acceptance gates on runtime speed; a
  candidate that is measurably faster is ACCEPTED even if the release binary grows
  (size is still measured and reported, but is not a rejection gate for a speed win).

The same codebase can be run twice — once per objective — and the two passes are
independent. A pass's report states the objective and the prompt set it used.

## Language detection

Determine the target codebase language from its manifest:

- `Cargo.toml` (or `src/**/*.rs`, a Cargo workspace) → **Rust**
- A `cpp/` directory plus a C++ build manifest (`CMakeLists.txt`, `meson.build`,
  `Makefile`, …) → **C++**

Only run prompts for a language whose directory exists in this repo. If the language
is not yet supported, say so and stop — do not invent prompts.

Supported languages and prompt locations:

- Rust → `rust/` (size/general) and `rust/speed/` (speed)
- C++ → `cpp/`

**Infrastructure concerns** (not a source language) are dispatched separately:

- **Docker** → `docker/` — whenever a `Dockerfile`, `*.Dockerfile`,
  `docker-compose.yml`, or `deploy/*.Dockerfile` is present, dispatch these prompts in
  addition to the source-language pass.
- **General / cross-cutting** → `general/` — dispatched on every run, alongside the
  source-language and Docker passes. These scan the whole repo for language-independent
  patterns (e.g. compression commands).

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
Rescan: run a second full pass on the optimized code
        ↓
        └─ repeat until a pass yields no newly accepted optimizations
        ↓
Produce final report
```

## Step 1 — Establish the baseline

Before touching code, record a reproducible baseline with the repo's own commands.

**Rust:**

```bash
cargo test                 # correctness baseline
cargo build --release      # release build
size <release-binary>      # or: stat -c %s, cargo bloat, cargo llvm-lines
cargo bench / hyperfine / time  # representative speed where available
```

Build the binary-size map with `cargo bloat --release --crates` and `--functions`,
`cargo llvm-lines`, `nm`, `objdump` where available.

**C++:**

```bash
cmake --build build --config Release     # or the repo's build command
ctest --test-dir build                   # correctness baseline
size <release-binary>                    # or: stat -c %s, nm -S --size-sort
hyperfine / time ./<binary> [args]       # representative speed
valgrind --leak-check=full --show-leak-kinds=all ./<binary>  # leak baseline
```

Build the binary-size map with `nm -S --size-sort <binary>`, `size -A <binary>`,
`bloaty`, or `objdump` where available.

Record exact commands and numbers so A/B comparisons stay comparable. Also record,
where practical: `.text`, `.rodata`, `.data`, allocation count/bytes, peak memory,
instruction count.

## Step 2 — Enumerate relevant source files

Build a **coverage ledger** covering every relevant source file for the detected
language:

```text
# Rust
src/**/*.rs
crates/**/src/**/*.rs
benches/**/*.rs
examples/**/*.rs
build.rs
tests/**/*.rs

# C++
src/**/*.{cpp,h,hpp,cc,cxx}
include/**/*.{h,hpp}
lib/**/*.{cpp,h,hpp}
test*/**/*.{cpp,h,hpp}

# Docker (infrastructure)
Dockerfile
**/*.Dockerfile
docker-compose.yml
**/docker-compose*.yml
.dockerignore

# General (cross-cutting)
**/*.sh
**/*.bash
Makefile
Justfile
.github/workflows/**/*.yml
**/*.yml
```

Exclude generated source, vendored dependencies, `target/`, `build/`, and external
submodules. A file is not "analyzed" merely because a search matched nothing — a
subagent must inspect its contents. The pass is not complete until coverage is 100%.

## Step 3 — Dispatch one subagent per prompt

Dispatch the source-language prompts, plus the `docker/` prompts when Dockerfiles are
present and the `general/` prompts on every run. For each prompt file in
`<language>/` (e.g. `rust/*.md`) when the objective is **size**, or in
`<language>/speed/` (e.g. `rust/speed/*.md`) when the objective is **speed** —
plus `docker/`, or `general/` — spawn a subagent whose job is to run **that single
prompt** against the codebase. Give each subagent:

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

**Balance across optimization dimensions.** Optimization spans many dimensions —
binary size, runtime speed, codegen, monomorphization, data layout, control flow,
algorithmics, AND heap allocations. Heap-allocation findings are easy to over-report;
do not let them crowd out the other dimensions. When a source-codebase's hot path is
already allocation-free (per-byte/per-item loops allocate nothing), say so and do not
manufacture low-value allocation candidates. Prioritize candidates that improve
multiple dimensions at once (less code AND fewer allocations AND less work).

**Full code is a hard gate.** Every candidate MUST arrive with COMPLETE
`Current code (before)` and `Proposed code (after)` — the full enclosing function,
method, or block (full signature + every statement) plus enough surrounding context
(types, imports, call sites) to apply verbatim without opening another file. No
ellipses, no truncated bodies, no single-line paraphrases, no prose standing in for
code. When a subagent returns a summary-only candidate or a fragment, the coordinator
MUST send it back to be completed — it must NOT reconstruct, infer, or accept it.

## Step 5 — A/B test every candidate individually

**Every viable candidate is A/B tested individually for BOTH speed AND release binary
size.** One candidate per experiment; never batch unrelated changes.
**A/B testing is MANDATORY for every candidate — HIGH, MEDIUM, and LOW** — no
candidate may be accepted from reasoning alone; it must be built and measured for
both size and speed. If the repo has no performance harness for the affected path,
the coordinator MUST provision one (a named microbenchmark or representative workload
with an exact command, e.g. `hyperfine 'sort large.txt'` / `perf stat`) rather than
leaving speed unmeasured. `unmeasured` is the exception, not the default.

For each candidate:

1. **Correctness gate first.** Apply only this candidate in isolation, then run the
   language-appropriate tests — `cargo test` for Rust, `ctest` (or the repo's test
   target) for C++ — plus workspace/integration/property tests as available. If it
   changes externally observable behavior, reject it.
2. **Release binary size.** Rebuild under identical conditions — `cargo build
   --release` for Rust, the release CMake/Make target for C++ — and record exact byte
   size. Binary-size deltas are deterministic given the same build environment.
3. **Speed.** Run the repo's benchmark or a representative microbenchmark around the
   affected path, multiple runs to beat noise. Use `cargo bench`, `hyperfine`,
   `perf stat`, or a C++ benchmark harness. Do not claim a runtime win from an
   unstable single run.
4. **Memory/leaks (C++ where relevant).** For memory-safety candidates, re-run the
   Valgrind/ASan baseline to confirm the leak count/bytes dropped.
4. **Allocations/memory** where the candidate targets them — measure allocation
   count/bytes if instrumentation exists.
5. **Docker (where relevant).** For Dockerfile candidates the metrics are **final
   image size** (`docker images <tag>`, or compressed via `docker save | gzip | wc -c`
   / `skopeo`), **build time** with a warm cache, and **layer count**; correctness is
   that the container still builds and passes its healthcheck. A/B each candidate
   individually as above.

Decide with these rules:

- **ACCEPT** when correctness unchanged AND binary smaller AND performance equal or
  faster.
- **ACCEPT** when correctness unchanged AND binary unchanged AND performance
  meaningfully faster.
- **ACCEPT** when correctness unchanged AND binary meaningfully smaller AND
  performance statistically unchanged.
- **TRADEOFF — REVIEW** when binary smaller but slower, or faster but larger. Report
  it explicitly; do not silently accept it.
- **Speed objective.** When the run's objective is **speed**, a candidate that is
  measurably faster is ACCEPTED even if the release binary grows — size is recorded
  but is not a rejection gate. Only report a TRADEOFF when the speed win is
  statistically weak or the size regression is extreme; otherwise the faster
  candidate is the accepted outcome of a speed pass.
- **REJECT** when it increases binary size, slows execution, increases allocations,
  breaks tests, worsens memory, or provides no measurable benefit. Record rejected
  experiments — they are useful information.

After each accepted candidate, rebuild the baseline and proceed to the next.

## Second pass — iterate to convergence

**One pass is not enough.** After every candidate in the first pass has been accepted
or rejected, apply a **second full optimization pass to the now-optimized code**. A
change accepted in one pass frequently reshapes code (APIs, ownership, generics,
async state, collections, parsers, hot paths) in ways that expose further
optimizations — and can revive candidates that were rejected earlier because their
prerequisite was absent. In practice a second pass on already-optimized code finds
additional measurable improvements.

Run each pass exactly like the first:

1. Re-establish the baseline on the current (already-optimized) code — tests,
   release binary size, speed — since measurements from the prior pass no longer
   reflect the working tree.
2. Re-enumerate and re-dispatch all prompts (Steps 2–3), because the changed code
   contains new sites and new opportunities.
3. Collect, classify, and deduplicate new candidates, then A/B test **every** new
   candidate individually for speed and release binary size (Step 5) against the new
   baseline.
**Two refinements are mandatory after every accepted candidate:**

- **Re-run the originating prompt over the change.** After a candidate is accepted
  and applied, run its prompt (and the related prompts it touches) scoped to the
  changed code, to find opportunities the change itself opened up. An edit frequently
  reshapes a hot path, removes an allocation, changes what a shared macro inlines, or
  alters collection/generic/async shape such that a previously-inapplicable
  optimization now applies — do not wait for the next full pass to find it.
- **Cross-apply the pattern to other similar sites.** When a prompt finds an
  optimization at one site, actively search the codebase for OTHER sites with the
  same underlying pattern — the same macro or helper, the same `.lock()` on a global
  I/O handle, the same buffer/collection idiom, the same error- or diagnostic-
  construction shape — that the prompt did not flag. The first hit is rarely the only
  instance; A/B test each cross-apply candidate individually and keep the ones that
  measure better. Cross-application is often where the cumulative win lives.

**Stop when a full pass produces zero newly accepted optimizations** (convergence).

**Stop when a full pass produces zero newly accepted optimizations** (convergence).
Do not keep looping forever — a pass with no new accepts is the natural stopping
point. If a second pass is not run, the report MUST say why (e.g. scope/time limit)
and the results are reported as partial.

The final report reports **cumulative** results across all passes: the true starting
baseline (pre-pass-1) through the final binary/benchmark, plus a per-pass breakdown
of candidates discovered/accepted/rejected. Rank accepted optimizations across all
passes by actual measured benefit.

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

For each candidate — accepted, rejected, or tradeoff — give: file, function, category,
COMPLETE `Current code (before)` and `Proposed code (after)` (full sections, never
fragments), reasoning, semantic validation, test status, measured binary-size delta,
measured speed delta, allocations delta, and ACCEPT/REJECT/TRADEOFF decision with
explanation. Rank accepted optimizations by actual measured benefit. Every candidate
in the report carries real before/after numbers for BOTH size and speed; do not omit
code or measurements from any candidate.

## Constraints

- This skill only optimizes the **source and build configuration** — do not change
  the contract, remove data checks, or introduce behavior changes.
- Never propose a change that removes or weakens a validation/data check.
- Do not add `unsafe` for optimization without strong measured evidence that safe Rust
  cannot achieve the same result, plus documented invariants and tests.
- Do not accept a plausible-looking change without A/B measurement. The target is not
  clever-looking code — it is measurably better code.
