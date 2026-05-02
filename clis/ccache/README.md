# ccache

> **A compiler cache that drops in front of `gcc` / `clang` /
> `nvcc` / MSVC and re-uses object files from previous compiles
> when the preprocessor output (or, in `direct_mode`, the
> source + headers + flags hash) matches — turning a clean
> rebuild from "minutes of `cc1` CPU" into "seconds of file
> copies", with a content-addressable on-disk store at
> `~/.cache/ccache/` (or `$CCACHE_DIR`) that is shareable
> across branches, worktrees, and via `secondary_storage` /
> `remote_storage` even across CI runners (HTTP, Redis,
> S3-compatible).** Pinned to **v4.13.4** (released 2026-04-19),
> [LICENSE.adoc](https://github.com/ccache/ccache/blob/v4.13.4/LICENSE.adoc),
> GPL-3.0-or-later.

Source: <https://github.com/ccache/ccache>

## TL;DR

`ccache` sits between your build system (`make`, `ninja`,
`cmake`, `bazel`, `meson`, `xcodebuild`) and the real
compiler. Install it, prepend `ccache` to `CC` / `CXX` (or
symlink `/usr/local/opt/ccache/libexec` into `PATH` so plain
`gcc` / `clang` resolve to ccache), and every subsequent
compile becomes a hash lookup against the cache; on a hit it
copies the cached `.o` (or hardlinks it with
`hard_link=true`) and skips the compiler entirely. The two
hit modes that matter:

- **direct mode** (default): hashes source + included
  headers + compile flags + compiler mtime. O(1), no
  preprocessor invocation. ~10× faster than preprocessor
  mode on a hit.
- **preprocessor mode** (fallback): runs the preprocessor,
  hashes the expanded output. Catches more hits when headers
  change in cosmetically-different ways but still slower than
  direct.

`ccache --show-stats` is the dashboard: cache hit ratio,
direct vs preprocessor hits, miss reasons, cache size on
disk. A clean rebuild of LLVM, Chromium, the Linux kernel,
or any C++ codebase with deep template instantiation goes
from "go to lunch" to "wait 30 seconds" once the cache is
warm.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ccache

# Debian / Ubuntu
sudo apt-get install ccache

# Fedora / RHEL
sudo dnf install ccache

# Static binary (Linux x86_64) — no toolchain pin
curl -fsSL -o /tmp/ccache.tar.xz \
  https://github.com/ccache/ccache/releases/download/v4.13.4/ccache-4.13.4-linux-x86_64-musl-static.tar.xz
tar -xf /tmp/ccache.tar.xz -C /tmp
sudo install /tmp/ccache-4.13.4-linux-x86_64-musl-static/ccache /usr/local/bin/

# Verify
ccache --version    # ccache version 4.13.4
```

Wire into a build:

```bash
# Option A — wrap explicitly
export CC="ccache gcc"
export CXX="ccache g++"

# Option B — symlink takeover (preferred for cmake / meson)
sudo ln -s /usr/local/bin/ccache /usr/local/bin/gcc
sudo ln -s /usr/local/bin/ccache /usr/local/bin/g++
sudo ln -s /usr/local/bin/ccache /usr/local/bin/cc
sudo ln -s /usr/local/bin/ccache /usr/local/bin/c++

# Option C — CMake native
cmake -DCMAKE_C_COMPILER_LAUNCHER=ccache \
      -DCMAKE_CXX_COMPILER_LAUNCHER=ccache -B build
```

## One Concrete Example

```bash
# Tune cache size + enable direct mode + compression
ccache --set-config=max_size=20G
ccache --set-config=compression=true
ccache --set-config=compression_level=6

# Cold build of a ~50k-LOC C++ project
time cmake --build build -j$(nproc)
# real    4m12s    user 32m08s    sys 1m44s

# Wipe object files, rebuild from scratch, watch ccache work
ninja -C build clean
time cmake --build build -j$(nproc)
# real    0m18s    user 0m44s     sys 0m11s

# Inspect the dashboard
ccache --show-stats
#   Cache hit (direct)        12,431 /  12,488    (99.5%)
#   Cache hit (preprocessor)      57 /  12,488    ( 0.4%)
#   Cache miss                     0 /  12,488    ( 0.0%)
#   Cache size (GB)            3.21 /  20.00     (16.05%)

# Share a cache across CI runners via S3-compatible store
ccache --set-config=remote_storage="s3://bucket/ccache?region=us-east-1"
ccache --set-config=remote_only=false
```

## License

[GPL-3.0-or-later](https://github.com/ccache/ccache/blob/v4.13.4/LICENSE.adoc),
SPDX `GPL-3.0-or-later`.

## Niche / positioning

The **canonical compiler cache** for C / C++ / Objective-C /
CUDA. Pick `ccache` over [`sccache`](../sccache/) when your
toolchain is purely native C/C++ on a single host, you want
the smallest dependency surface (a single C++ binary with no
runtime), and the GPL is acceptable for build-time tooling
(it is — ccache never links into your output, it only caches
its files). Pick [`sccache`](../sccache/) instead when you
need Rust support, native cloud storage backends as
first-class, or Apache-2.0 for corporate-policy reasons.
Orthogonal to [`bazel`](../bazel/) / [`buck2`](../buck2/) —
those have their own remote caches; ccache is for projects
*not* on a hermetic build system. Skip when builds are already
< 30s (overhead exceeds savings) or when the language is Rust
(use `sccache`), Go (`go build` already caches), or
JVM (`gradle --build-cache`).
