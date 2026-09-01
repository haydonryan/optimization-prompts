# optimization-prompts

A prompt library for a single, language-agnostic **optimization skill**.

Instead of maintaining a separate skill for every optimization concern (allocation
elimination, ownership/clone cleanup, monomorphization shrink, SIMD vectorization,
type-system hardening, …) — and then duplicating each one for Rust, C++, and every
future language — this repo keeps those specialist prompts in one place. The optimization skill reads the relevant prompt for the task at hand and runs it.

## Size vs speed objectives

Each language root carries **two prompt sets**, selected by the run's objective:

- `<language>/` (e.g. `rust/`) — the **general/size** set. Acceptance gates on release
  binary size; speed must not regress.
- `<language>/speed/` (e.g. `rust/speed/`) — the **speed** set. Acceptance gates on
  runtime speed; a candidate that is measurably faster is accepted even if the binary
  grows (size is still measured and reported, but is not a rejection gate for a speed
  win).

Every coordinator mode takes an **objective** (`size` or `speed`, default `size`) and
dispatches the matching set. The same codebase can be run once per objective; the two
passes are independent. A speed pass on a hot path targets inlining, cache locality,
allocation elimination, vectorization, branch predictability, algorithmic work,
hashing, I/O batching, concurrency, parsing/formatting, iteration, and hoisting
invariant setup — each measured with runtime as the primary criterion.

**The repository is just prompts. The skill is the runner.** The skill owns the
baseline/A-B measurement loop, correctness validation, and final report; each prompt
here is one focused optimization pass the skill can dispatch.

The coordinator skill lives at [`SKILL.md`](SKILL.md), with two variant modes:
[`SKILL-stories.md`](SKILL-stories.md) (turns every A/B-verified improvement into a
backlog story via `add-backlog-story`) and [`SKILL-report.md`](SKILL-report.md)
(writes a complete detailed report including LOW-priority candidates). Each mode
detects the language, spawns one subagent per prompt in the matching language
directory, then classifies and A/B tests every discovered candidate individually for
**speed** and **release binary size**.

## Why this exists

- **One skill, not dozens.** Every optimization *category* and *language* combination
  is a file here, not a new skill that must be installed, versioned, and kept in sync.
- **Prompt improvements land as normal PRs.** Editing a prompt is a code review, not a
  skill reinstall.
- **The optimizer contract stays in one place.** Measurement discipline (A/B test every
  candidate, preserve observable behavior) lives in the skill; the pass-specific
  knowledge lives here.

## Modes

Three coordinator skills share the same prompt library and differ only in output.
Each mode also takes an **objective** (`size` or `speed`) that selects the prompt set
it dispatches (see the size vs speed objectives section above):

- **`SKILL.md`** — base mode. Runs every prompt, A/B tests each candidate, and
  produces the final report of accepted optimizations.
- **`SKILL-stories.md`** — backlog-story mode. Runs every prompt, A/B tests each
  candidate across the **LTO on/off × x86_64/aarch64** matrix, and turns every
  accepted improvement into a backlog story via the `add-backlog-story` skill. No
  improvement lands without a full A/B matrix row.
- **`SKILL-report.md`** — detailed-report mode. Runs every prompt, A/B tests each
  candidate, and writes a complete report that **includes LOW-priority candidates**
  (they are never dropped) — each with flagged code, proposed remediation, per-arch
  A/B evidence, and verification.

## Repository layout

Prompts are organized first by language, then by optimization category.

```text
optimization-prompts/
├── SKILL.md              # the optimization coordinator skill (root)
├── SKILL-stories.md      # coordinator variant: A/B (LTO × arch) → backlog stories
├── SKILL-report.md       # coordinator variant: complete report incl. LOW candidates
├── rust/                 # concern-specific prompts, one per optimization concern
│   ├── allocation-elimination.md
│   ├── async-future-state.md
│   ├── bounds-checks-iteration.md
│   ├── codegen-flags.md
│   ├── collections-data-structures.md
│   ├── compact-idiomatic-code.md
│   ├── control-flow-generated-code.md
│   ├── data-structure-optimization-audit.md
│   ├── dependency-feature-reduction.md
│   ├── dry-code-reduction.md
│   ├── hasher-selection.md
│   ├── initialization-buffer-reuse.md
│   ├── inlining-cold-code.md
│   ├── io-syscall-reduction.md
│   ├── logging-diagnostics.md
│   ├── memory-retained-peak.md
│   ├── monomorphization.md
│   ├── ownership-clone-copy.md
│   ├── parallelism-exploitation.md
│   ├── release-profile-binary-size.md
│   ├── repeated-work-algorithmic.md
│   ├── serialization.md
│   ├── simd-vectorization.md
│   ├── string-formatting-parsing.md
│   ├── structural-review.md
│   ├── struct-enum-layout.md
│   ├── synchronization.md
│   └── speed/           # speed-focused prompts (objective = speed)
│       ├── algorithmic.md
│       ├── allocation-elimination.md
│       ├── branch-prediction.md
│       ├── cache-locality.md
│       ├── concurrency.md
│       ├── hashing.md
│       ├── inlining.md
│       ├── io.md
│       ├── iteration.md
│       ├── lazy-initialization.md
│       ├── parsing-formatting.md
│       └── vectorization.md
├── cpp/                 # C++ prompts
│   └── ownership-raii.md  # memory leaks + manual ownership → RAII
├── docker/              # infrastructure prompts (dispatched when Dockerfiles present)
│   └── dockerfile-minimization.md  # multi-stage, minimal runtime image
└── general/             # cross-cutting prompts (dispatched on every run)
    └── compression-level.md  # compression commands without an explicit level
```

Language roots exist only when they have at least one prompt. A category that applies
to a language gets its own file under that language's directory; there is **no** shared
cross-language directory. If the same technique applies to Rust and C++, write it twice,
tuned to each language's idioms — do not generalize into a hybrid prompt. Likewise, a
concern that has both a size and a speed angle gets two files: the general pass under
`<language>/` and the speed-tuned pass under `<language>/speed/` (see the size vs
speed objectives section above).

## Prompt format

Every prompt is a single Markdown file with YAML frontmatter, followed by the
instructions the skill should feed to its analysis pass.

```markdown
---
name: allocation-elimination
language: rust
category: heap-allocation
applies_to: src/**/*.rs
---

# Heap Allocation Elimination

Inspect every file for `String`, `Vec`, `Box`, `format!`, `collect::<Vec<_>>()`, ...
```

Required frontmatter fields:

| Field       | Meaning                                                              |
|-------------|----------------------------------------------------------------------|
| `name`      | Stable, unique prompt id (also the filename stem).                    |
| `language`  | Target language: `rust` (or later `cpp`).                            |
| `category`  | Optimization concern this prompt covers (e.g. `heap-allocation`).     |
| `applies_to`| File glob(s) the pass should scan, or `*` for whole-repo passes.      |

Keep `applies_to` honest: whole-repo passes (e.g. type-system hardening) use `*`;
targeted passes (e.g. SIMD hot loops) restrict scope.

## How the skill uses this repo

1. The skill detects the target language from the build manifest (e.g. `Cargo.toml`).
2. It establishes a baseline: tests, release binary size, and representative speed.
3. It spawns **one subagent per prompt** in the matching language directory, each
   running its prompt against the codebase and returning classified candidates.
4. The skill deduplicates candidates, then **A/B tests each one individually** for
   speed and release binary size (correctness gate first).
5. It accepts/rejects each candidate and produces the final report.

Every prompt enforces a shared invariant regardless of concern:

- Each found optimization is **classified** (category, file, site, semantic risk,
  expected impact).
- Each candidate includes **concrete before/after code** (not prose) and names the
  **specific verification test / scenario** that guards the change.
- Each is **A/B tested individually** for speed and release binary size; `Measured
  impact` records real before/after numbers — measurement is mandatory, never
  speculative.
- Correctness and 100% relevant-source coverage are always required.

## Adding a prompt

1. Create `rust/<category>.md` (or the matching language directory).
2. Fill in the frontmatter and write the pass instructions.
3. Keep it self-contained: a pass must be runnable with just the skill's baseline and
   the text of the prompt. Reference the skill's invariants rather than redefining them.
4. Open a PR. The prompt is reviewed like code.

## License

[MIT](LICENSE)
