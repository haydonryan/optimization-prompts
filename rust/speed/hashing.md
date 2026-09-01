---
name: hashing
language: rust
category: speed-hashing
applies_to: "*"
---

# Hashing and Map Selection for Speed

You are the **hashing-for-speed** specialist pass for a Rust speed-optimization run.
Your objective is to cut the cost of hashing and map/set operations on the hot path.
Rust's default `HashMap` uses SipHash-1-3, which is DoS-resistant but slow for
high-throughput workloads; the right hasher or structure can be several times faster
with no behavioral change when ordering and hash-flooding resistance are not
required.

## Scope

Inspect every relevant Rust source file plus `Cargo.toml` dependencies. Focus on hot
lookup / insert paths: per-item dedup, counters, caches, memoization, joins, and any
`HashMap`/`HashSet`/`BTreeMap` used inside loops or per record.

## What to look for

- **Default `HashMap`/`HashSet` (SipHash) on a hot lookup/insert path** where the keys
  are not attacker-controlled and ordering is irrelevant — switching to `ahash`,
  `fxhash` (`FxHashMap`), or another fast hasher (or `BuildHasherDefault`) is a direct
  speed win. Flag the DoS / determinism tradeoff: do **not** swap hashers when keys
  are attacker-controlled or hashing must be stable across runs.
- **A map used only for iteration-order or membership** where a `BTreeMap` (ordered)
  or a plain sorted `Vec` + binary search would be faster or allocation-free.
- **Hashing the same key repeatedly** per iteration — hoist the hash or the lookup out
  of the loop, or reuse the computed `Hash`.
- **A hot path building a `HashMap` from a small fixed set** where a match/array/lookup
  table or a `Vec` indexed by an enum discriminant is cheaper and allocation-free.
- **String keys with hashing cost** — `&str` vs `String` keys, interning repeated
  keys, or using a cheaper key type.
- **Re-hashing on every access** where the map could be built once and reused.

Prefer the simplest change (a fast hasher or a better structure) with the same
behavior. Only switch to `unsafe`-based hashing with strong measured evidence. A
candidate is a win if the hot path is measurably faster; report size too but do not
reject it for growing the binary.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: speed-hashing-<N>
Title:
File:
Function / site:
Category: speed-hashing
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
  for code. If the change adds a dependency or a `use`, include that line. A candidate
  whose before/after omits any part of the affected code is INCOMPLETE — the
  coordinator MUST reject it rather than reconstruct it.
- **Measurement is mandatory — runtime speed first, binary size still recorded —
  never speculative.** Every candidate carries real before/after numbers for BOTH:
    - **runtime speed** via the repo's benchmark harness if one exists, otherwise a
      concrete named microbenchmark / representative workload with the exact command
      (`hyperfine 'insert 1M keys'`, `perf stat`, `cargo bench`), many repetitions to
      beat noise; and
    - **release binary size** via `ls -l target/release/<bin>` / `cargo bloat`.
  **Decision criterion for this speed pass: runtime speed.** A faster candidate is
  ACCEPTED even if the binary grows — size is reported but is not a rejection gate.
  Do NOT default to `unmeasured`.
- **Measure on the target architectures.** Report each arch's numbers (size via
  cross-compile, speed natively or flagged `qemu-*` emulation). If an arch cannot be
  measured, mark it `unmeasured` and name the missing toolchain.
- **`Verification / test` names the specific existing test or scenario** that guards
  the change, plus the command to run it (e.g. `just test && just check`). If the key
  type, ordering, or equality semantics change, name a test that pins the behavior.
- **De-duplicate across passes.** If another prompt already reported the same
  candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead of a duplicate
  block.

Each candidate is A/B tested individually by the coordinator for **speed** (primary)
and **release binary size** (reported, not a rejection gate) — correctness gate first.
Give enough detail that the change can be applied in isolation.
