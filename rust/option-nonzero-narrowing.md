---
name: option-nonzero-narrowing
language: rust
category: memory-layout
applies_to: "*"
---

# Option<u8> → Option<NonZeroU8> Niche Narrowing

You are the **niche narrowing** specialist pass for an exhaustive Rust optimization
run. Your objective is to find `Option<u8>` values that can be replaced with
`Option<NonZeroU8>` to shrink their memory footprint.

## Why it helps

`u8` has no niche, so `Option<u8>` needs an extra tag byte: `size_of::<Option<u8>>()`
is **2 bytes**. `NonZeroU8` uses the forbidden `0` as its niche, so
`Option<NonZeroU8>` is **1 byte** with no padding and no separate tag. Replacing one
with the other therefore **halves the footprint of every such value**. The win matters
where many instances exist and pack tightly: struct fields on hot/oft-instantiated
types, `Vec<Option<u8>>` elements, and arrays of them. The generated code also drops
the tag load/store.

## The hard correctness guardrail (MUST enforce)

The replacement is **only valid where `0` is never a meaningful `Some` value** at the
site. `Some(0)` must never be produced or consumed meaningfully. Concretely, before
marking any candidate you MUST verify all of:

- No assignment or construction path creates `Some(0)` (e.g. `Some(x)` where `x` can
  be `0`), and no consumer reads the value expecting a distinct `0`.
- The value's domain does not include `0` as a valid meaning — examples that INVALIDATE
  a candidate: an hour of day (0 = midnight), a port, an index/count that can be zero,
  a status/type code where `0` is a variant, a CLI "disable" sentinel where
  `Some(0)` means "0 disables", a hex-nibble parser that yields `Some(0)` for `b'0'`,
  a rating/speed/level where `0` is a real value.
- The value does **not** cross a serialization/deserialization, database, network,
  CLI-arg, or FFI boundary where `0` and `None` must serialize as distinct values.
  `Option<u8>` and `Option<NonZeroU8>` have **different serde representations**
  (`Option<NonZeroU8>` serializes `None` as `null` and cannot represent `0` at all),
  so any schema/DB/API boundary change must be a deliberate, reviewed break — never an
  incidental one.

If any of the above holds, the site is NOT a candidate. Prefer rejecting a borderline
site over widening scope: the niche win is small, and a mistaken conversion that turns
a valid `Some(0)` into `None` (or a serialization error) is a silent correctness bug.

## Scope

Inspect every relevant Rust source file for `Option<u8>` (also catch `Option::<u8>`,
and `Option<u8>` nested inside generic parameters such as `Vec<Option<u8>>`,
`HashMap<K, Option<u8>>`, or `[Option<u8>; N]`). Prioritize:

- struct fields and enum payloads (highest value — they shrink the whole type)
- `Vec`/array/collection element types (dense footprint)
- function parameters and returns only when they are the storage type of a hot value,
  not when they merely pass a `u8` around
- `u8` fields/collections already using the niche convention elsewhere in the file
  (consistency signal)

## Survey evidence (HNR-Global Rust repos, 2026-11-01)

A read-only sweep of all 23 Rust repos that have tracker-cli projects found **26
`Option<u8>` sites total and 0 applicable candidates**:

- **fastclaw** (11): delete-message-days, Discord `channel_type` (0 = GUILD_TEXT),
  gateway JSON int params, hex-nibble `Some(0)`, speech-detector level 0 — all
  meaningful-0 and/or serde/CLI/serde_json boundaries.
- **fastdrone** (5): drain-cron window hour fields (0 = midnight), serde config/status.
- **photos** (3): `rating` fields on serde request structs persisted to a DB column
  where `None` = "no update" (COALESCE) but `0` is a valid rating — Some(0) vs None is
  semantically required.
- **strobe** (3): clap CLI arg fields where `Some(0)` is the "disable" sentinel
  (value_parser ranges 0..=8 / 0..=7).
- **argy** (2): test-only CLI-option demo structs; `0` a valid speed value.
- **vaultwarden-cli** (2): serde `r#type` (0 = valid item type), hex-digit `Some(0)`.

Repos with **zero** `Option<u8>`: candle, catalyst, matrix-notifier, finance, checkd,
workout-tracker, status-page, frigate-rust-migration, chatterbox-tts-rs, fastcut,
events, freshrss-rust-migration, cargo-autopatch, couchdb-file-sync, host-watchd,
open-stt-server.

Verdict: the technique is real and worth keeping as a prompt, but in our current
codebase there is no live conversion to apply — the pattern tends to appear exactly
where `0` is a meaningful value or at a serde/CLI/DB boundary, so it is a **low-yield,
high-risk** pass. Run it only when a type genuinely cannot hold `0` and does not cross
a serialization boundary.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it. If a site was considered
but rejected (0 valid, or a boundary), you do not need to report it; only report
candidates that pass the guardrail.

```text
Candidate ID: option-nonzero-narrowing-<N>
Title:
File:
Function / site:
Category: memory-layout
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
  test exists, say so and specify the regression to add — at minimum, assert
  `size_of::<T>()` shrank and that `Some(0)`-producing and boundary paths still
  behave (or are intentionally removed).
- **De-duplicate across passes.** If another prompt already reported the same
  candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead of a duplicate
  block.

Each candidate is A/B tested individually by the coordinator for **speed** and
**release binary size** (correctness gate first). Give enough detail that the change
can be applied in isolation.
