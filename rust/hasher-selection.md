---
name: hasher-selection
language: rust
category: hasher-selection
applies_to: "*"
---

# Hash Function / Hasher Selection Optimization

You are the **hasher selection** specialist pass for an exhaustive Rust
optimization run. Your objective is to find `HashMap`/`HashSet`/`Hash` uses where the
default SipHash hasher is unnecessary and a faster hasher (for non-adversarial keys)
or a cheaper key type would speed up lookups and reduce allocations.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits. Look at every `HashMap`, `HashSet`, `Hash`, `BTreeMap` (which needs no hasher),
and any `contains`/`get`/`insert` hot path.

## What to look for

Look for:

- `HashMap`/`HashSet` with default (`RandomState`/SipHash) used on **non-adversarial,
  non-attacker-controlled** keys in hot paths — consider a faster hasher such as
  `fxhash` (`FxHashMap`), `ahash`, or `rustc-hash` when the input is not a security
  boundary (a hash-flooding attack is the only reason to keep SipHash)
- maps keyed by dense integers or enums that could be a `Vec<Option<T>>`, an array,
  or a bitset instead of hashing at all
- `String` keys that could be `&str`, interned IDs, or typed keys to avoid hashing the
  whole allocation and cloning it
- keys whose `Hash` impl is expensive (e.g. hashing a whole `PathBuf`/`String`/large
  struct repeatedly instead of a cheap derived ID)
- `BTreeMap` used where a `HashMap` (faster, unordered) would do and ordering is not
  actually relied upon — but do **not** switch if iteration order is externally
  observable (serialized output, stable iteration, tests asserting order)
- duplicate hashing (hash the same key, re-derive it, or clone it just to look up)

**Security caveat:** if the map is reachable from untrusted/network input where an
adversary could craft colliding keys (HTTP headers, JSON keys, auth data), keep the
default SipHash or an explicitly randomized hasher. Only propose a faster hasher when
you can state why the keys are not attacker-controlled.

Do **not** swap hashers speculatively — a faster hasher is not always faster for
small key sets, and it adds a dependency. Prefer the standard library / a tiny
`BuildHasher` implementation and measure.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: hasher-selection-<N>
Title:
File:
Function / site:
Category: hasher-selection
Priority: HIGH | MEDIUM | LOW
Description:
Current code (before):
Proposed code (after):
Measured impact (per target arch: binary size / speed, before -> after):
Verification / test:
Semantic risk (what could break / what to verify):
```

Requirements:

- **Code is mandatory — the FULL section, not a fragment.** `Current code (before)`
  and `Proposed code (after)` MUST each contain the COMPLETE enclosing function,
  method, or block — the full signature plus every statement/expression — together
  with enough surrounding context (relevant types, imports, and call sites) that the
  coordinator can apply the change verbatim without opening any other file. No
  ellipses (`...`), no truncated bodies, no single-line paraphrases, no prose standing
  in for code. If the change spans a whole type, module, or Cargo/profile block, show
  the complete before and after of that unit. A candidate whose before/after omits any
  part of the affected code is INCOMPLETE — the coordinator MUST reject it rather than
  reconstruct it.
- **Measurement is mandatory for BOTH binary size AND runtime performance — never
  speculative.** Do not write "may improve speed" or "may reduce size". Every candidate
  carries real before/after numbers for BOTH:
    - **release binary size** via `ls -l target/release/<bin>` / `cargo bloat` /
      `cargo llvm-lines`; and
    - **runtime performance** via the repo's benchmark harness if one exists, otherwise
      a concrete named microbenchmark or representative workload you specify with the
      exact command (e.g. `hyperfine 'sort large.txt'`, `perf stat`, `time`), run enough
      repetitions to beat noise.
  Record both in `Measured impact`. Do NOT default to `unmeasured`: every candidate must
  have a named performance measurement and a real before/after number. If a candidate
  genuinely has no measurable runtime path (e.g. a pure build/profile-flag change), say
  so explicitly and give the size result with that note. If the coordinator cannot
  measure in this environment, it marks the candidate `unmeasured` and names exactly
  which measurement is missing — but that is the exception, not the default.
- **Measure on the target architectures.** A/B results are architecture-specific:
  layout, alignment, `target-cpu`, and vectorization differ between `x86_64` and
  `aarch64`. Build and measure on every arch the software ships on (commonly both).
  Binary size is measurable for any arch via cross-compilation
  (`cargo build --target <triple>`); runtime speed requires native execution (or
  `qemu-*` emulation, flagged as such). Report the arch each number came from; if an
  arch cannot be measured, mark it `unmeasured` and name the missing toolchain.
- **`Verification / test` names the specific existing test or scenario** that guards
  the change, plus the command to run it (e.g. `just test && just check`). If no
  test exists, say so and specify the regression to add.
- **De-duplicate across passes.** If another prompt already reported the same
  candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead of a duplicate
  block.

Each candidate is A/B tested individually by the coordinator for **speed** and
**release binary size** (correctness gate first). Give enough detail that the change
can be applied in isolation.
