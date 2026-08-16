---
name: io-syscall-reduction
language: rust
category: io-syscall-reduction
applies_to: "*"
---

# I/O and Syscall Reduction Optimization

You are the **I/O / syscall reduction** specialist pass for an exhaustive Rust
optimization run. Your objective is to reduce the number and cost of syscalls,
subprocess spawns, filesystem operations, and network round-trips.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits. Look at every `std::fs`, `std::io`, `Command`, `process`, `TcpStream`,
`reqwest`, websocket, and serialization/deserialization-from-file site.

## What to look for

Look for:

- repeated `Command`/subprocess spawns whose result is stable (e.g. re-running
  `lsblk`, `systemctl`, `which`, or a CLI probe on every loop iteration instead of
  caching the parsed result)
- `read_to_string`/`read` of a whole file into memory then parsing (peaks at ~2x
  file size; prefer streaming reads when the file is large)
- `fs::write` / `rename` churn: writing a full serialized state on every tick even
  when nothing changed
- unbuffered `File`/`TcpStream` reads and writes (no `BufReader`/`BufWriter`) causing
  many small syscalls
- per-call network client construction (new `reqwest::Client`, new TLS handshake, no
  connection reuse / keep-alive)
- no read/write timeouts, so a single hung descriptor stalls the loop
- opening the same file or directory repeatedly, path resolution recomputed, missing
  `create_dir_all` consolidation
- full-file buffering where streaming would do (large temp `Vec<u8>`/`String` held in
  memory)
- missing batch/flush discipline on `Write` (many tiny writes instead of one
  buffered flush)
- `statvfs`/`stat`/metadata calls that could be cached for the poll lifetime

Prefer safe, boring changes: buffer I/O, reuse a connection, cache stable results,
avoid spawning a subprocess when the parse can be cached, and skip writes when
nothing changed. Do **not** trade away correctness (atomic write-tmp-rename must stay
atomic; flush before rename; preserve error paths and timeouts).

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: io-syscall-reduction-<N>
Title:
File:
Function / site:
Category: io-syscall-reduction
Priority: HIGH | MEDIUM | LOW
Description:
Current code (before):
Proposed code (after):
Measured impact (per target arch: binary size / speed, before -> after):
Verification / test:
Semantic risk (what could break / what to verify):
```

Requirements:

- **Code is mandatory.** `Current code (before)` and `Proposed code (after)` MUST be
  concrete, apply-able code snippets (Rust unless stated otherwise) — not prose
  summaries. If a candidate genuinely cannot be expressed as code, say so explicitly
  and show the surrounding call site.
- **Measurement is mandatory, not speculative.** Do not write "may improve speed" or
  "may reduce size". Apply the change in isolation and measure: release binary size
  via `cargo bloat` / `ls -l target/release/<bin>` (or `cargo llvm-lines` where
  relevant), and speed via the repo's own benchmark/profiling harness if one exists
  (or count syscalls with `strace -c` / `/usr/bin/time -v`). Record real before/after
  numbers in `Measured impact`. If you cannot measure in this environment, mark the
  candidate `unmeasured` and name exactly which measurement is missing.
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
