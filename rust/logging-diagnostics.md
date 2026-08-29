---
name: logging-diagnostics
language: rust
category: logging-diagnostics
applies_to: "*"
---

# Logging and Diagnostics Optimization

You are the **logging / diagnostics** specialist pass for an exhaustive Rust
optimization run. Your objective is to find logging and diagnostic code that does
expensive work even when the log is disabled.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits.

## What to look for

Inspect:

```rust
trace!  debug!  info!  warn!  error!
```

and custom diagnostic systems. Look for:

- strings formatted before log-level checks
- allocations only needed for logs
- expensive calculations performed when logging is disabled
**Global I/O-handle `.lock()` bloat (fat-LTO specific).** Look for
`std::io::stderr().lock()` / `stdout().lock()` used for a **single** diagnostic or
output write, especially in shared macros/helpers invoked by many functions (e.g. a
`show_error!`/`show_warning!`/`show!` macro). Under `lto = "fat"` (or "thin"), the
`StderrLock`/`StdoutLock` guard type — its `Write` impl and its unlock+flush `Drop`
machinery — is inlined into **every** call site, so N diagnostic-emitting functions
each carry a few KB of per-site guard code. Removing the `.lock()` and writing
directly to `stderr()`/`stdout()` (which take the internal reentrant lock per write)
routes every call site through one compact shared path, shrinking the binary by
per-site-guard × N sites. Verified real-world example: removing `.lock()` from the
`show_*!`/`show!` diagnostic macros of uutils/coreutils shrank the fat-LTO binary by
~110 KB across ~155 functions.

**Safety rule for this pattern:** only remove `.lock()` where the handle is locked for
a **single** write (or writes whose atomic grouping is not load-bearing). Do NOT
remove it where the lock is held across a loop / multi-write batch / stream, or where
the caller relies on holding the lock (that is intentional and removing it changes
atomicity or adds per-write locking). For each candidate, name whether it is a
single-write site (safe) or a held-lock site (keep), and A/B measure size and speed.

Do **not** remove useful observability solely for size — optimize how it is produced

Do **not** remove useful observability solely for size — optimize how it is produced
(e.g. lazy argument evaluation with closures like `ok_or_else`, `with_context`, or
log-enabled gating).

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: logging-diagnostics-<N>
Title:
File:
Function / site:
Category: logging-diagnostics
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
