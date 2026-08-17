---
name: simd-vectorization
language: rust
category: simd-vectorization
applies_to: "*"
---

# SIMD and Auto-Vectorization Optimization

You are the **SIMD / vectorization** specialist pass for an exhaustive Rust
optimization run. Your objective is to find loops that LLVM can auto-vectorize (or
that need a nudge to do so) and byte/numeric processing that can run several
elements per instruction.

## Scope

Inspect every relevant Rust source file. Scan actual file contents, not just search
hits. Pay special attention to numeric, byte-scanning, parsing, checksum, and
formatting loops. Use `cargo asm`/`objdump` or `rustc -C llvm-args=...` to confirm
whether a loop actually emits SIMD (`xmm`/`ymm`/`zmm`, `padd`, `pmul`, etc.).

## What to look for

Look for tight loops over `u8`/`u16`/`u32`/`u64`/`f32`/`f64` slices, matrices, and
arrays where each iteration does independent work:

```rust
for i in 0..bytes.len() {
    acc += bytes[i];
}
```

Check for:

- loops LLVM refuses to vectorize due to aliasing, carried dependencies, or
  irregular access (strided, gathered, or data-dependent indices)
- `#[inline]`-inhibiting, branch-heavy, or function-call-per-element loops
- `iter()`/`collect()`/`sum()` pipelines that build intermediate `Vec`s instead of
  a fused vectorizable reduction
- byte scanners, base64/hex/URL decode, `percent()`, `human_bytes`, checksums,
  hashing, and `u64`/`f64` transformations over large inputs
- matrix/linear-algebra and cross-correlation loops
- `-C target-cpu=native` / `target-feature` being disabled when the target CPU
  supports SIMD
- `#[repr(align(N))]` and `#[repr(packed)]` that prevent aligned vector loads

Prefer safe, ordinary Rust that auto-vectorizes. Do **not** reach for explicit
`unsafe` `std::arch` intrinsics unless safe Rust provably cannot produce the
vectorized code and the measured win justifies it. When a loop only vectorizes for
one element type, consider `simd`-friendly restructure or runtime feature dispatch
(`is_x86_feature_detected!`) rather than unconditional intrinsics.

Do **not** propose `-C target-cpu=native` blindly for a distributable binary — it
can break portability to older CPUs. Flag the portability tradeoff and prefer a
runtime-dispatched or conservative baseline unless the binary is machine-pinned.

## Output contract

Report every candidate in this exact format. Do **not** modify the codebase — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: simd-vectorization-<N>
Title:
File:
Function / site:
Category: simd-vectorization
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
