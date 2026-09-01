---
name: concurrency
language: rust
category: speed-concurrency
applies_to: "*"
---

# Concurrency and Contention Reduction for Speed

You are the **concurrency-for-speed** specialist pass for a Rust speed-optimization
run. Your objective is to use available cores on independent hot work and to remove
contention that serializes it. When the hot path is single-threaded but the work is
embarrassingly parallel, or when threads stall on a contended lock, the fix is a
measurable latency/throughput win.

## Scope

Inspect every relevant Rust source file. Focus on hot paths that process independent
work: per-record/per-request loops, batch transforms, fan-out over collections, and
any shared state (`Mutex`, `RwLock`, `Arc`, atomics, channels) touched on the hot
path. Use `perf stat` (context switches), `perf lock`, or `cargo flamegraph` to see
where threads block.

## What to look for

- **A sequential loop over independent items** that could be parallelized with
  `std::thread::scope`, `rayon` `par_iter`/`par_chunks`, or split ranges — but only
  when the per-item work is large enough to amortize the split overhead and the result
  order can be preserved.
- **A contended `Mutex`/`RwLock`/global lock on the hot path** where the critical
  section can be shortened, sharded (per-key / per-shard locks), or replaced with
  atomics (`AtomicUsize`, `fetch_add`, `compare_exchange`) — only when the lock was
  protecting a counter or a small cell, not a real invariant.
- **False sharing** — distinct hot fields of different threads sharing one cache line
  (adjacent fields in one struct, or adjacent array elements written by different
  threads), causing coherence traffic. Fix with padding (`#[repr(align(64))]`,
  `CachePadded`) or by giving each thread its own slot.
- **Lock granularity too coarse** (one lock for a whole table) where fine-grained or
  lock-free read paths would scale.
- **Channels / async task hops on the hot path** adding latency where a direct call or
  a batch would do.
- **Oversubscription or contention on a fixed pool** where the workload is
  I/O-bound — do not add threads to an already-starved path.

Prefer safe, idiomatic parallel primitives (`rayon`, `std::thread::scope`, atomics)
over `unsafe` lock-free code. A concurrency candidate is a win if throughput/latency
is measurably better and correctness (ordering, determinism) is preserved; report size
too but do not reject a faster parallel path for growing the binary.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: speed-concurrency-<N>
Title:
File:
Function / site:
Category: speed-concurrency
Priority: HIGH | MEDIUM | LOW
Description:
Current code (before):
Proposed code (after):
Measured impact (per target arch: speed, before -> after; binary size, before -> after):
Verification / test:
Semantic risk (what could break / what to verify):
```

Requirements:

- **Code is mandatory — the FULL section, not a fragment.** `Current code (before)`
  and `Proposed code (after)` MUST each contain the COMPLETE enclosing function,
  method, or block (full signature + every statement/expression) together with enough
  surrounding context (relevant types, imports, and call sites) to apply verbatim
  without opening another file. No ellipses, no truncated bodies, no prose standing in
  for code. If the change spans a whole type or module, show the complete before and
  after of that unit. A candidate whose before/after omits any part of the affected
  code is INCOMPLETE — the coordinator MUST reject it rather than reconstruct it.
- **Measurement is mandatory — runtime speed first, binary size still recorded —
  never speculative.** Every candidate carries real before/after numbers for BOTH:
    - **runtime speed** (throughput or latency) via the repo's benchmark harness if one
      exists, otherwise a concrete named microbenchmark / representative workload with
      the exact command (`hyperfine 'transform 10M rows'`, `perf stat`, `cargo bench`),
      many repetitions to beat noise; and
    - **release binary size** via `ls -l target/release/<bin>` / `cargo bloat`.
  **Decision criterion for this speed pass: runtime speed / throughput.** A faster
  candidate is ACCEPTED even if the binary grows — size is reported but is not a
  rejection gate. Do NOT default to `unmeasured`.
- **Measure on the target architectures.** Parallelism and atomics differ between
  `x86_64` and `aarch64` (memory ordering, cache coherence); report each arch's numbers
  (size via cross-compile, speed natively or flagged `qemu-*` emulation). If an arch
  cannot be measured, mark it `unmeasured` and name the missing toolchain.
- **`Verification / test` names the specific existing test or scenario** that guards
  the change, plus the command to run it (e.g. `just test && just check`).
  Concurrency changes MUST be validated for determinism/ordering and data races (run
  the test under `cargo test` and, where feasible, `--release` or thread sanitizer).
- **De-duplicate across passes.** If another prompt already reported the same
  candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead of a duplicate
  block.

Each candidate is A/B tested individually by the coordinator for **speed** (primary)
and **release binary size** (reported, not a rejection gate) — correctness gate first.
Give enough detail that the change can be applied in isolation.
