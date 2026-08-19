---
name: compression-level
language: general
category: compression-level
applies_to: "*"
---

# Compression Level Specification

You are the **compression-level** specialist pass for an exhaustive optimization run.
Your objective is to find every place a compression command is invoked **without an
explicit compression level** (e.g. `gzip` with no `-9`), and flag each one for A/B
testing between the default level and **maximum compression**, measuring both **size**
and **speed** (compression and decompression).

This is a cross-cutting, language-independent concern. Run it on the whole repo — it
is not tied to a single source language.

## Scope

Scan every place compression commands are invoked, including but not limited to:

- shell / build scripts (`*.sh`, `*.bash`, `Makefile`, `Justfile`)
- CI config (`.github/workflows/*.yml`, `.gitlab-ci.yml`, Jenkinsfiles, Travis,
  CircleCI)
- deploy / packaging scripts (`Dockerfile`, `docker-compose.yml`, release/archive
  steps, `*.spec`, `PKGBUILD`, `cargo package`/`npm pack` hooks)
- backup / log-rotation scripts (`logrotate`, cron jobs)
- any `Command`/`Process`/`std::process`/`subprocess` invocations that shell out to a
  compression tool
- in-process compression library calls where a level parameter is omitted or defaults
  (e.g. `gzip`/`zlib`/`flate2`/`brotli`/`zstd` with no `Compression`/`level` argument)

Scan actual file contents, not just search hits.

## What to look for

Compression commands invoked **without an explicit level flag**:

```sh
gzip file.tar              # default (-6) — flag: no level
tar czf out.tgz dir/       # -z with no gzip level
zip -r out.zip dir/        # default compression
bzip2 file                 # default (-9 is bzip2's max, but default is still worth verifying)
xz file                    # default (-6) vs -9e
zstd file                  # default (3) vs --ultra -22
pigz file                  # default
7z a out.7z dir/           # default (-5)
tar cJf out.txz dir/       # xz default
tar --zstd -cf out.tzst dir/
compress file
```

For each invocation record:

- the exact command and where it is called
- the default level the tool applies (e.g. gzip `-6`, xz `-6`, zstd `3`, 7z `-5`)
- the maximum level available for that tool (gzip/bzip2/pigz `-9`, xz `-9e`, zstd
  `--ultra -22`, 7z `-mx=9`, zip `-9`)
- the data being compressed (file type, typical size) so the A/B workload is
  representative

Do **not** change the command yourself — report it as a candidate. The coordinator
runs the A/B.

## A/B testing (size and speed)

Every flagged invocation is A/B tested individually by the coordinator. For each, run
the tool twice on representative data: once at its **default level**, once at
**maximum compression** (e.g. `gzip -9`, `xz -9e`, `zstd --ultra -22`, `7z -mx=9`).
Measure, with multiple runs to beat noise:

- **Compressed size** — bytes before/after; the size reduction is the primary win.
- **Compression time** — wall time to produce the archive; maximum levels are often
  dramatically slower to compress.
- **Decompression time** — wall time to extract; usually a smaller win but still
  measure it.
- **Correctness** — decompressed output must be byte-identical to the input.

Report the explicit tradeoff: maximum compression shrinks the artifact but costs
compression time (and marginally decompression time). Whether the size win is worth
the speed cost depends on the workload (one-off release artifact vs. on-the-fly
compression in a hot path). Do not blindly accept `-9` on a hot path that compresses
frequently and rarely stores.

## Output contract

Report every candidate in this exact format. Do **not** modify the files — the
coordinator applies each change in isolation and A/B tests it.

```text
Candidate ID: compression-level-<N>
File:
Command / site:
Category: compression-level
Tool: (gzip / zip / xz / zstd / 7z / bzip2 / other)
Current implementation: (command as written, default level)
Proposed implementation: (command with max level)
Data compressed: (file type / typical size, so A/B workload is representative)
Why it may reduce stored size / bandwidth:
Why it may impact speed (compression and decompression):
Semantic risk (what could break / what to verify — e.g. tool availability of max level):
Expected impact: HIGH | MEDIUM | LOW
```

Each candidate is A/B tested individually by the coordinator for **compressed size**
and **compression + decompression speed**, with the tradeoff reported explicitly.
Give enough detail that the change can be applied in isolation.
