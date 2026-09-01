---
name: unnecessary-locks
language: rust
category: synchronization
applies_to: "*"
---

# Unnecessary Locks

You are the **unnecessary-locks** specialist pass for an exhaustive Rust optimization
run. Your objective is to find locks — `Mutex`, `RwLock`, and the atomics/`Arc` used
around them — that waste time on hot paths because they are redundant, oversized,
wrong-primitive, or dead, while preserving every concurrency guarantee.

## Relationship to other passes

This pass is narrower and lock-specific. It complements `synchronization.md` (which
covers `Arc`/`Rc`/reference-count churn broadly) and `hasher-selection.md`. If a
candidate is really about eliminating an `Arc` clone or choosing a map type, emit a
cross-reference (`dup of <prompt>-<N>`) instead of duplicating it here. This pass owns
the lock-and-ordering anti-patterns below.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits. Before judging any lock, establish the **thread model**: which OS threads touch
each piece of shared state (e.g. a per-output render thread vs. a single main/event-loop
thread), and whether the guarded data is cross-thread, single-threaded, or already
serialized by an enclosing lock. A lock is only "unnecessary" if removing it keeps the
program correct under that thread model.

## What to look for

Inspect every `Mutex`, `RwLock`, `Atomic*`, and `parking_lot`/`std::sync` usage. Look
for:

- **Redundant inner locks under an already-held outer lock.** A `Mutex`/`Atomic` field
  inside a type that is only ever reached through an enclosing `Arc<Mutex<T>>` or
  `RwLock` guard (e.g. a widget/program struct whose every accessor already locks the
  outer handle) adds per-access cost and no exclusion — the outer lock is the sole
  serialization point. The inner lock can become a plain field. Verify no access path
  bypasses the outer lock before proposing this.
- **Redundant with an enclosing higher-level lock.** Per-element/per-object
  `Arc<Mutex<Option<T>>>` fields where every writer holds an enclosing exclusive lock
  (`RwLock::write`) and every reader an enclosing shared lock (`RwLock::read`). If no
  writer ever runs under a bare shared lock, the inner `Mutex` is redundant; Copy
  guarded types can become a cheap atomic, or a plain `Cell`/field after a full audit.
- **Write lock taken when a read suffices.** A `RwLock::write()` (or exclusive `Mutex`
  path) whose body only reads. Common on hot paths. Check the callee's signature: if
  it takes `&T`, a write lock is wrong.
- **Wrong primitive for the access pattern.** A `Mutex` guarding read-mostly,
  single-writer state (many reads per frame, rare writes) where `parking_lot::RwLock`,
  an `AtomicUsize`/`AtomicBool`/`AtomicU64` (for Copy values), or `ArcSwap` would make
  the hot readers lock-free or uncontended. A `std::sync::Mutex` (fair, heavier) where
  `parking_lot::Mutex` would suffice.
- **Set-once state behind a lock + `Option`.** A `Arc<Mutex<Option<T>>>` assigned
  exactly once at init before any reader can run, where `OnceLock<T>`/`OnceCell<T>`
  removes the lock and the unwrap.
- **Single-threaded state behind a lock.** A `Mutex`/`RwLock` whose only readers and
  writers run on the same thread (verified by call-site audit), where `RefCell`/`Cell`
  or a plain field removes the atomic acquire/release entirely. `UserDataMap`
  non-threadsafe slots and event-loop-only state are common candidates.
- **Dead or write-only synchronization.** An `AtomicBool`/`Mutex` field that is only
  ever stored to, never loaded (write-only), or read in a path that can never observe
  the write. Delete the field and the store.
- **Repeated locking of the same lock in one function.** Two or more acquisitions of
  the same lock back-to-back (or across a loop) that could be one hold — but never hold
  a lock across a call that can re-enter or deadlock, and respect existing
  deadlock-avoidance drops.
- **Unconditional locking on a path that is a no-op.** A lock taken on every event
  even when the feature it guards is disabled (e.g. a cursor auto-hide mutex locked on
  every input event when auto-hide is off). Early-return before locking when the work
  is provably a no-op.
- **`SeqCst` where a weaker ordering suffices.** `Ordering::SeqCst` on pure
  trigger/flag/one-shot atomics whose data is otherwise synchronized by an enclosing
  lock or a release/acquire hand-off. `Relaxed`, `Acquire`, or `Release` removes the
  full fence on per-frame/per-event loads.
- **Oversized critical section.** A lock held across slow work (allocation, I/O, GL
  calls, network) when only a small field needs guarding — narrow the scope or split
  the locked data.

Preserve **all** concurrency guarantees: correctness under the real thread model,
atomicity, ordering, and lock invariants. Never remove a lock that protects genuine
cross-thread mutation. When in doubt, name the thread-model evidence that proves the
lock is (or is not) needed rather than guessing.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: unnecessary-locks-<N>
Title:
File:
Function / site:
Category: synchronization
Priority: HIGH | MEDIUM | LOW
Thread model (who writes / who reads, read vs write ratio):
Description:
Current code (before):
Proposed code (after):
Measured impact (per target arch: binary size / speed, before -> after):
Verification / test:
Semantic risk (what could break / what to verify):
```

Requirements:

- **Establish the thread model first.** Every candidate must state the observed
  writers, readers, and threads touching the guarded data (with `File:line` evidence),
  and whether the lock is cross-thread, single-threaded, or redundant with an outer
  lock. A candidate without this evidence is INCOMPLETE.
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
      exact command (e.g. `hyperfine`, `perf stat`, `time`), run enough repetitions to
      beat noise. For a lock removal on a hot path (render loop, per-event handler),
      name the specific loop/event and how its cost is measured.
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
  test exists, say so and specify the regression to add. For a lock change, call out
  any concurrency test (stress, race, multi-threaded) that exercises the touched path.
- **De-duplicate across passes.** If `synchronization.md` or another prompt already
  reported the same candidate, emit a cross-reference (`dup of <prompt>-<N>`) instead
  of a duplicate block.

Each candidate is A/B tested individually by the coordinator for **speed** and
**release binary size** (correctness gate first). Give enough detail that the change
can be applied in isolation.
