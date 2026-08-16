# optimization-prompts

A prompt library for a single, language-agnostic **optimization skill**.

Instead of maintaining a separate skill for every optimization concern (allocation
elimination, ownership/clone cleanup, monomorphization shrink, SIMD vectorization,
type-system hardening, …) — and then duplicating each one for Rust, C++, and every
future language — this repo keeps those specialist prompts in one place. The
optimization skill reads the relevant prompt for the task at hand and runs it.

**The repository is just prompts. The skill is the runner.** The skill owns the
baseline/A-B measurement loop, correctness validation, and final report; each prompt
here is one focused optimization pass the skill can dispatch.

The coordinator skill lives at [`SKILL.md`](SKILL.md). It detects the target
language, spawns one subagent per prompt in the matching language directory, then
classifies and A/B tests every discovered candidate individually for **speed** and
**release binary size**.

## Why this exists

- **One skill, not dozens.** Every optimization *category* and *language* combination
  is a file here, not a new skill that must be installed, versioned, and kept in sync.
- **Prompt improvements land as normal PRs.** Editing a prompt is a code review, not a
  skill reinstall.
- **The optimizer contract stays in one place.** Measurement discipline (A/B test every
  candidate, preserve observable behavior) lives in the skill; the pass-specific
  knowledge lives here.

## Repository layout

Prompts are organized first by language, then by optimization category.

```text
optimization-prompts/
├── SKILL.md              # the optimization coordinator skill (root)
├── rust/                 # concern-specific prompts, one per optimization concern
│   ├── allocation-elimination.md
│   ├── ownership-clone-copy.md
│   ├── collections-data-structures.md
│   ├── monomorphization.md
│   ├── string-formatting-parsing.md
│   ├── async-future-state.md
│   ├── struct-enum-layout.md
│   ├── data-structure-optimization-audit.md
│   ├── control-flow-generated-code.md
│   ├── bounds-checks-iteration.md
│   ├── repeated-work-algorithmic.md
│   ├── initialization-buffer-reuse.md
│   ├── inlining-cold-code.md
│   ├── synchronization.md
│   ├── serialization.md
│   ├── logging-diagnostics.md
│   ├── structural-review.md
│   ├── compact-idiomatic-code.md
│   ├── dry-code-reduction.md
│   ├── memory-retained-peak.md
│   ├── dependency-feature-reduction.md
│   └── release-profile-binary-size.md
└── cpp/                 # future
    └── ...
```

Language roots exist only when they have at least one prompt. A category that applies
to a language gets its own file under that language's directory; there is **no** shared
cross-language directory. If the same technique applies to Rust and C++, write it twice,
tuned to each language's idioms — do not generalize into a hybrid prompt.

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
- Each is **A/B tested individually** for speed and release binary size.
- Correctness and 100% relevant-source coverage are always required.

## Adding a prompt

1. Create `rust/<category>.md` (or the matching language directory).
2. Fill in the frontmatter and write the pass instructions.
3. Keep it self-contained: a pass must be runnable with just the skill's baseline and
   the text of the prompt. Reference the skill's invariants rather than redefining them.
4. Open a PR. The prompt is reviewed like code.

## License

[MIT](LICENSE)
