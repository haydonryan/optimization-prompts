---
name: data-structure-optimization-audit
language: rust
category: memory-layout
applies_to: "*"
---

# Rust Data-Structure Optimization Audit

Find bloated, sloppy, over-sized, allocation-heavy, cache-unfriendly, or
unnecessarily complex data structures and propose changes that reduce:

* in-memory structure size
* heap allocations
* pointer indirection
* padding/alignment waste
* duplicated state
* unnecessary capacity
* oversized integer/discriminant types
* unnecessary ownership
* cache misses
* serialization/deserialization overhead where applicable

This is a **report-first** audit. You identify and stack-rank opportunities; you do
**not** implement them. A/B testing happens later, in the implementation stories you
hand off (see "End state" below).

## Critical safety requirements

Do **not** make changes that break or silently alter:

* existing on-disk formats
* serialized data formats
* database schemas
* network protocols
* IPC formats
* public APIs
* FFI/ABI layouts
* memory-mapped structures
* binary compatibility
* stable configuration formats
* previously persisted files
* backwards compatibility expectations

Before suggesting a structural change, determine whether the structure participates
in any external representation. Look for: `serde`, `bincode`, `postcard`, `rkyv`,
`prost`, `tonic`, protobuf, JSON, YAML, TOML, MessagePack, CBOR, custom
encoders/decoders, `Read`/`Write` impls, `From`/`Into` conversions to wire types,
database row mappings, SQLx types, FFI types, `#[repr(...)]`, mmap, unsafe
transmutation, raw byte casts, file headers, binary formats, snapshots/golden files,
public crate types.

If changing a structure could alter an external format, do **not** recommend the
incompatible change directly. Instead, separate the internal representation from the
persistent/wire representation:

```rust
// Stable external representation
struct DiskRecord { id: u64, flags: u32 }

// Optimized internal representation
struct Record { id: RecordId, flags: Flags }
```

Conversion boundaries are preferable to breaking an existing format. Check every
serialization path explicitly; "size_of is smaller" is never enough on its own when a
type is serde-boundary or persisted.

## Ordering semantics (hard guard)

Before changing any collection type, iteration order, or index-based operation, treat
**ordering** as load-bearing until proven otherwise. An ordering change is a
correctness change, not a micro-optimization.

For any ordered→unordered swap (e.g. `BTreeMap`→`HashMap`, `Vec`→`HashSet`,
sorted `Vec`→`HashMap`) or order-affecting operation (`swap_remove`, `retain`,
`sort` → `sort_unstable`, `dedup` that changes multiplicity), verify ALL of:

- whether iteration order is **externally visible** (serialized output, logs,
  notification text, stdout — any format emitted to a user or another system)
- whether callers **keep indexes** or rely on positional access into the collection
- whether **iteration order matters later** in the same or a downstream pass
- whether **tests pin ordering** (assert exact output or sequence)

**Especially: anything read directly from disk in order.** If the structure is
populated from a file — persisted state, a config, a log, or a record stream that is
read (or written) sequentially and whose on-disk order is significant — do not
reorder or re-represent it in a way that changes the disk format or the order in
which records are read/written. Preserve the on-disk representation and its read
order even if an unordered in-memory representation would be faster; only change the
internal representation behind a conversion boundary that keeps the disk format
byte-identical and the read order unchanged.

If you cannot prove ordering is irrelevant, do not propose the change; say so and
mark it `ordering-uncertain`.

## Phase 0 — Calibrate to actual scale (do this first)

Before analyzing any structure, establish the real scale of the codebase so the
audit's priorities match reality. This is a required gate, not a nicety.

1. Inventory the codebase: every Rust file, every struct/enum/collection, and which
   structures are hot (materialized in large counts or long-lived) vs. singleton /
   transient.
2. Find **evidence** of population sizes: migrations, seed data, README/deploy
   config, benchmarks, profiling output. Look for the repo's own profiling or
   benchmark harness (e.g. `dhat`, `criterion`, `valgrind`/`massif`, a `bench`/
   `examples` profile target) and **run it** to ground allocation and peak-memory
   claims.
3. State a realistic population for each candidate. If you cannot find evidence,
   label the estimate `[INFERENCE]` and **do not** let a guessed population drive the
   ranking. A 40-byte win across 5 million instances matters; the same win across 500
   instances is trivia — and many real repos are the latter.
4. Set expectations accordingly. Do not import "millions of instances" framing into a
   small codebase. The aggregate lens (`bytes × population`) is the correct ruler,
   but the populations must come from the codebase, not from the template's examples.

This phase shapes the final tiering: a change that is correct but touches a singleton
belongs in Tier 3/4 no matter how elegant it is.

## Phase 1 — Audit the entire codebase

Inspect **every** Rust source file. Do not stop after a few obvious cases. Build an
inventory of: structs, enums, tuples used as records, collections, nested
collections, maps/sets, strings, buffers, optional fields, boxed values,
reference-counted values, serialized records, temporary intermediate representations,
caches, queues, AST/data-model nodes, request/response types.

Pay particular attention to structures created in large numbers or retained for long
periods.

### What to look for

1. **Struct padding / field ordering** — reorder fields to cut padding; measure with
   `size_of`/`align_of`. Do not reorder if layout is externally significant. Weight
   by population.
2. **Oversized integer types** — `usize`/`u64`/`i64`/`u32` where the domain is much
   smaller. Do **not** narrow just because current test data fits; determine the real
   semantic range. Consider changing internal representation while preserving the
   serialized one.
3. **Bloated enums** — a single large variant inflates every value. Consider boxing
   rare large variants, but account for allocation, locality, and variant frequency.
   Measure before/after; do not box mechanically.
4. **`Option<T>` representation** — know the real memory cost; exploit niche
   optimization (`Option<NonZeroU32>`, `Option<&T>`, `Option<Box<T>>`). Don't introduce
   obscure representations for trivial gains.
5. **Excessive `String`** — fixed-vocabulary / repeated / immutable strings may be
   enums, interned, `Arc<str>`, `Box<str>`, `Cow`, borrowed, or typed IDs. Preserve
   external string representations when serialized.
6. **`String` vs `Box<str>`** — immutable long-lived strings can drop the capacity
   word. Rank higher when there are many instances.
7. **Repeated strings/identifiers** — interning or typed IDs, only when cardinality ×
   repetition justifies the complexity.
8. **Excessive `Vec<T>`** — size ~0/1, small fixed max, compile-time-known, or never
   mutated → `Box<[T]>`, `[T; N]`, `SmallVec`, `ArrayVec`, `Option<T>`. Consider
   actual size distribution and mutation.
9. **`Vec`/`String`/`HashMap` capacity waste** — bad initial estimates, retained
   capacity after spikes. Don't recommend churn-y shrinking.
10. **Tiny-cardinality collections** — hash/btree maps with a handful of elements may
    be better as `Vec<(K,V)>`, sorted Vec, or arrays. Measure lookup frequency.
11. **Wrong collection for workload** — `HashMap` for tiny maps, `BTreeMap` without
    ordering need, `VecDeque` for append+iterate, maps keyed by dense ints → `Vec<Option<T>>`,
    bitsets, arrays, sorted Vecs, arenas.
12. **Boolean-heavy structs** — padding inflating size; consider bitflags only when
    savings justify reduced readability. Preserve serialization.
13. **Duplicate/derivable fields** — e.g. a `bool` that mirrors `is_empty()`, or a
    filename derivable from a path. Remove only if not an intentional cache.
14. **Duplicate ownership** — same data cloned into multiple structures; consider
    move/borrow/share/index/intern. Avoid hard lifetimes for trivial savings.
15. **Excessive `Arc`/`Rc`** — check whether shared ownership is truly needed;
    prefer `Arc<str>`/`Arc<[T]>` over `Arc<String>`/`Arc<Vec<T>>`.
16. **Excessive `Box`** — `Vec<Box<SmallStruct>>` is poor density; analyze rather
    than mechanically unboxing.
17. **Nested allocations** — `Vec<Vec<T>>`, `Vec<String>`, `HashMap<K, Vec<V>>` at
    high scale; estimate allocations per logical object; consider flattening/arenas.
18. **Struct-of-arrays** — large homogeneous collections with hot single-field loops.
    Architectural; require strong evidence.
19. **Hot/cold splitting** — rarely-accessed substantial fields out of hot structs
    into boxed cold data / side tables. Account for allocation + indirection.
20. **Rare fields bloating every instance** — move into optional boxed cold
    structures or side tables only when population and size justify it.
21. **Path representation** — repeated/prefix-shared `PathBuf`/`OsString`/`String`;
    be conservative about platform-specific path behavior.
22. **IDs as heavyweight objects** — repeated `String`/`Uuid`/`PathBuf` identity →
    dense internal IDs with a central table. Preserve external format via conversion.
23. **Enum representation/discriminants** — measure `size_of`; don't add `#[repr(u8)]`
    blindly (Rust niche layout can beat manual representation).
24. **Alignment amplification** — high-align fields inflating nested structures
    across big arrays.
25. **Zero-sized/phantom fields** — verify they're genuinely ZST and not adding
    wrapper complexity.
26. **Temporary intermediate structures** — multi-stage pipelines that materialize
    duplicate populations; consider streaming / in-place / buffer reuse.
27. **Representation conversion chains** — `String -> PathBuf -> String`,
    `Vec<T> -> HashSet<T> -> Vec<T>`, JSON → intermediate → final; remove redundant
    intermediates.
28. **Sparse structures** — many `Option<T>` fields where an enum better models
    mutual exclusivity — only if semantics actually enforce it.
29. **Redundant indexes/caches** — multiple lookup representations; justify each by
    lookup frequency; don't remove useful indexes to save memory.
30. **Global memory amplification** — always compute `bytes × population`; this
    dominates stack ranking.

## Measure instead of guessing

For important candidates, measure `size_of`/`align_of` **and** heap/allocation
behavior:

* current size, proposed size, bytes saved, percentage reduction
* estimated population (with evidence or `[INFERENCE]`), aggregate savings
* allocation count, capacity, heap bytes, nested allocations
* **run the repo's existing profiling harness** (e.g. `dhat`) on a representative
  workload to ground allocation and peak-RSS claims; if none exists, say so rather
  than fabricating numbers

`size_of` alone is insufficient for heap-backed structures. Where you cannot
measure, be explicit that the claim is unmeasured.

## A/B testing is deferred to implementation

This audit does **not** implement changes, so it cannot run before/after A/Bs itself.
Resolve that tension explicitly:

* Provide the **current baseline** as precisely as you can (measured `size_of`,
  profiled allocations where the harness exists).
* For each Tier 1/2 recommendation, specify the **A/B to run during implementation**:
  the metric (`size_of`, allocation count, peak RSS, serialization bytes) and the
  acceptance threshold. Do not assume a smaller struct is automatically faster.
* Reject changes whose added complexity or likely runtime regression outweighs the
  realistic benefit, even if they shrink memory.

## Compatibility verification

For anything serialized or persisted, verify the optimization does not change the
external representation. Where practical, specify compatibility tests: old fixture →
new code → equivalent value; new code → serialized bytes → exact expected
representation. For binary formats compare byte-for-byte; for text compare semantic
and (where stability matters) exact output. Existing fixtures and golden tests must
continue to pass — encode that as acceptance criteria for the handoff.

## Stack-rank every finding

Do not produce a flat list. Assign each finding a score:

* **Memory impact** — 5 very large aggregate saving … 1 negligible
* **Runtime benefit** — 5 likely major cache/allocation/runtime win … 1 likely slower
* **Confidence** — 5 strongly demonstrated … 1 speculative
* **Implementation complexity** — 1 trivial … 5 architectural
* **Compatibility risk** — 1 effectively none … 5 likely externally incompatible

Approximate priority:

```text
Priority =
    (MemoryImpact * 4)
  + (RuntimeBenefit * 2)
  + (Confidence * 2)
  - (ImplementationComplexity * 2)
  - (CompatibilityRisk * 4)
```

Treat this formula as a **triage heuristic, not truth**. The weights double-penalize
compatibility risk and complexity, which may be over-weighted for small internal
services with a tiny API surface. If the ordering looks wrong for the actual
codebase (e.g. the risk of a compat break is cheap because the API is internal), say
so and adjust with justification. Population evidence (or its absence) may override
the arithmetic.

## Scale-aware output format

Give the full per-finding template only to Tier 1 and Tier 2 findings. Minor /
workload-dependent findings (Tier 3/4) get a compact one-liner: location, structure,
problem, proposed, and why it was deprioritized. Do not inflate trivial findings with
the full 14-field template.

### Full template (Tier 1/2)

```text
Rank:
Location:
Structure:
Current representation:
Observed problem:

Current size:
Estimated heap usage:
Estimated population:        (evidence or [INFERENCE])
Estimated aggregate cost:

Proposed representation:

Expected size:
Expected memory saving:
Expected runtime effect:
Expected allocation reduction:
A/B to run during implementation:   (metric + acceptance threshold)

File-format / API compatibility:
Compatibility strategy:

Memory impact: X/5
Runtime benefit: X/5
Confidence: X/5
Implementation complexity: X/5
Compatibility risk: X/5

Priority score:
Evidence:
Recommendation:
```

Include concrete source locations (e.g. `src/index/node.rs:47`).

### Compact format (Tier 3/4)

```text
Location: <file:line>
Structure: <type>
Problem: <one line>
Proposed: <one line>
Deprioritized because: <one line>
```

## Final stack-ranked summary

At the beginning of the final report, provide a table similar to:

| Rank | Candidate             | Current |     Proposed | Est. Saving | Population | Aggregate Saving | Risk   | Priority |
| ---- | --------------------- | ------: | -----------: | ----------: | ---------: | ---------------: | ------ | -------: |
| 1    | Node metadata         |    96 B |         64 B |        32 B |       2.1M |           ~67 MB | Low    |       31 |
| 2    | Event enum            |   144 B |         32 B |       112 B |       300K |           ~34 MB | Medium |       27 |
| 3    | Repeated path strings |       — | interned IDs |           — |       1.8M |           ~21 MB | Low    |       25 |

Then divide the findings into:

* **Tier 1 — Do these first:** large measurable improvement, strong confidence, low
  compatibility risk.
* **Tier 2 — Strong candidates:** meaningful improvement, more validation/work.
* **Tier 3 — Situational:** smaller wins or workload-dependent changes.
* **Tier 4 — Not recommended:** investigated and rejected (insignificant savings,
  worse performance, too complex, or too risky). **Always include rejected ideas** —
  this prevents future engineers from re-evaluating changes already dismissed.

## End state

The deliverable is the audit report only. For Tier 1/2, hand off each finding as a
separately-shippable unit (its own story/change) whose acceptance criteria encode the
compatibility checks and the A/B metric from this report. Do not implement the
changes in this pass; do not leave a half-modified codebase.

## Guiding principle

We are not trying to produce the cleverest or smallest possible Rust representation.
We are trying to find **memory amplification and structural slop that matters at real
scale** — where "real scale" comes from the codebase itself, not from assumptions.

Prefer:

```text
large aggregate saving
+ fewer allocations
+ better cache locality
+ simple implementation
+ zero compatibility breakage
```

over clever representations that save a handful of bytes but make the code harder to
maintain. A 16-byte saving in an object instantiated five million times is important.
A 16-byte saving in an object instantiated twice is not.

**Find the multiplication factors — with evidence.**
