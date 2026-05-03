# mold

> **Modern, drop-in replacement linker (`ld` / `lld` / `gold`) that
> links large C / C++ / Rust binaries 5–10× faster by parallelising
> symbol resolution and section copy across all CPU cores and using
> a careful in-memory layout that avoids the random I/O patterns
> traditional linkers fall into.** Pinned to **v2.41.0**, MIT
> ([LICENSE](https://github.com/rui314/mold/blob/main/LICENSE)).

- **Repo:** https://github.com/rui314/mold
- **Latest version:** v2.41.0
- **License:** MIT (`LICENSE` at repo root)
- **Category:** `build-tooling` / `linker` / `compile-time-perf`
- **Language:** C++

## What it does

`mold` is a linker — the program the compiler driver invokes after
`cc1` / `rustc` / `clang` has produced object files, to stitch
`.o` archives plus dynamic libraries into a final ELF (Linux),
Mach-O (macOS, via the `ld64`-compatible front-end), or PE
(Windows, experimental) executable. The bottleneck in modern
incremental dev loops on a large C++ / Rust codebase is rarely the
compiler — it is the linker, because every `cargo build` /
`ninja` rerun re-links the whole binary even when only one
translation unit changed. `mold` rewrites the linker from scratch
around three observations: (1) symbol resolution is embarrassingly
parallel if you shard the symbol table by hash; (2) section copy
is bandwidth-bound, so it should use `mmap` + `memcpy` across all
cores rather than serial `read` / `write`; (3) the merge step for
identical-content sections (ICF) can be done in parallel with a
content-addressed hash. The result on a typical workstation: a
1.5 GB Chromium debug link drops from ~30 s with `gold` /
~12 s with `lld` to ~2 s with `mold`. For a Rust binary with
heavy generics (anything depending on tokio + serde + axum), a
`cargo build` rebuild after touching one file goes from ~6 s
linker time to under 1 s.

## Install

```bash
# macOS — note: macOS Mach-O support is experimental; on macOS
# most users still use the system ld64 / lld. mold's primary
# target is Linux.
brew install mold

# Linux (most distros)
sudo apt install mold        # Debian / Ubuntu (recent)
sudo dnf install mold        # Fedora
sudo pacman -S mold          # Arch
```

## Examples

```bash
# Use mold for a one-off Rust build without changing project config
RUSTFLAGS="-C link-arg=-fuse-ld=mold" cargo build --release

# Make mold the default for a Rust project: add to .cargo/config.toml
cat >> .cargo/config.toml <<'EOF'
[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]
EOF

# Use mold for a CMake / ninja C++ build
cmake -DCMAKE_LINKER=mold -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=mold" ..
ninja
```

## Why it matters in an AI-native workflow

The fastest feedback loop wins. When an agent (Claude Code, Codex,
Aider, Cline, Crush — all already cataloged in this zoo) is
iterating on a Rust or C++ codebase, the dominant wall-clock cost
between "agent edits a file" and "agent sees the test result" is
the link step, not compilation (`cargo` / `ninja` cache compiled
TUs aggressively, but always re-link). Cutting linker time from
12 s to 2 s on every iteration turns a 60-step agent debugging
session from 12 minutes of dead time into 2 minutes — a 6× cycle
speedup with zero changes to source code or build system. `mold`
is the single most leveraged install for any agent-driven Rust
or C++ workflow. Pairs cleanly with [`sccache`](../sccache/)
(compile cache, orthogonal — sccache cuts re-compile, mold cuts
re-link), [`cargo-nextest`](../cargo-nextest/) (parallel test
runner, so the saved link time actually becomes saved test time),
and [`bacon`](../bacon/) (file-watcher that re-runs cargo on save
— with mold each save-rerun is 5× cheaper). Orthogonal to
[`ccache`](../ccache/) which caches the compile step C++ side.
