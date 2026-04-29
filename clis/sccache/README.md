# sccache

> **Shared compilation cache for C / C++ / CUDA / Rust** — a
> drop-in `ccache` replacement that supports local disk *and*
> shared object-storage backends (S3, GCS, Redis, memcached,
> Azure Blob, OSS, R2), so a CI fleet or a multi-developer team
> can warm one cache and reuse compiled object files across
> machines. Pinned to **v0.14.0** (SPDX: `Apache-2.0`,
> [LICENSE-APACHE](https://github.com/mozilla/sccache/blob/main/LICENSE-APACHE)).

Source: <https://github.com/mozilla/sccache>

## TL;DR

`sccache` is what you put in front of `cc` / `c++` / `nvcc` /
`rustc` to make repeated compilations free. Locally it behaves
like `ccache` (hash the preprocessed input + flags, cache the
resulting `.o`). The differentiator is the *shared* backends:
point all your CI runners at the same S3 bucket and a `cargo
build` that took 4 min on a cold runner takes 20 s on the next
runner that touches the same source tree. Also supports
distributed compilation via `sccache-dist`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install sccache

# Cargo
cargo install --locked sccache

# Pre-built binary
curl -Lo sccache.tar.gz "https://github.com/mozilla/sccache/releases/download/v0.14.0/sccache-v0.14.0-aarch64-apple-darwin.tar.gz"
tar xf sccache.tar.gz
sudo install sccache-v0.14.0-aarch64-apple-darwin/sccache /usr/local/bin/

# Verify
sccache --version    # sccache 0.14.0
```

## License

Apache-2.0 — see
[LICENSE-APACHE](https://github.com/mozilla/sccache/blob/main/LICENSE-APACHE).

## One Concrete Example

```bash
# 1. local-disk cache (default), wrap your compiler
export RUSTC_WRAPPER=sccache
export CC="sccache cc"
export CXX="sccache c++"
cargo build --release

# 2. inspect cache stats (hits / misses / size)
sccache --show-stats

# 3. zero the stats counter (does NOT delete the cache)
sccache --zero-stats

# 4. stop the background server (it will respawn on next call)
sccache --stop-server

# 5. point the cache at S3 instead of local disk
export SCCACHE_BUCKET=my-team-sccache
export SCCACHE_REGION=us-west-2
export SCCACHE_S3_KEY_PREFIX=ci/
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export RUSTC_WRAPPER=sccache
cargo build

# 6. point the cache at Redis (cheap, low-latency on a LAN)
export SCCACHE_REDIS=redis://cache-host:6379/
sccache --show-stats     # shows the active backend

# 7. cap the local disk cache at 20 GiB
export SCCACHE_DIR=$HOME/.cache/sccache
export SCCACHE_CACHE_SIZE=20G

# 8. CI: print stats at the end of the job for visibility
sccache --show-stats >> $GITHUB_STEP_SUMMARY 2>&1 || true
```

## Niche It Fills

**One compilation cache that the whole team / CI fleet shares.**
`ccache` is the classic local cache and is excellent on a single
machine. `sccache` adds the *cloud-shared* backend that turns
ephemeral CI runners (each starting cold) into a single warm
cache pool. For Rust specifically it is the canonical
`RUSTC_WRAPPER` choice.

## When to use

1. **You run a Rust workspace in CI on ephemeral runners.**
   `cargo build` on a cold runner is the typical 5–15 min CI
   hit; pointing `RUSTC_WRAPPER=sccache` at an S3 bucket cuts
   it to seconds for unchanged crates.
2. **You have a C++ monorepo with multiple developers / agents
   recompiling the same translation units.** Shared S3 / Redis
   backend lets every machine pull the prior `.o` instead of
   recompiling.
3. **You want a single tool that handles `cc`, `c++`, `nvcc`,
   and `rustc`.** `ccache` does not cache `rustc`; `sccache`
   does both.
4. **You are willing to operate the backing store.** S3 lifecycle
   policy, Redis maxmemory, etc., are your responsibility.

## When NOT to use

- **Single developer on one machine, C/C++ only, no Rust.**
  `ccache` is older, more battle-tested, and slightly faster on
  the local-only path. Use `sccache` only if you also need Rust
  or the shared backend.
- **You compile with link-time optimisation across the entire
  binary every build.** LTO kills cache effectiveness because
  the cached unit is the post-LTO object that depends on every
  input. `sccache` will silently miss most of the time.
- **Your builds depend on absolute paths embedded in `__FILE__`
  or debuginfo.** Cached `.o` files baked on machine A will
  carry machine A's source paths. Use `--remap-path-prefix`
  (Rust) or `-fdebug-prefix-map` (C/C++) to make builds
  reproducible-by-path first.
- **You cannot operate the backing store.** A misconfigured S3
  bucket leaks proprietary intermediate object code. Use a
  private bucket with bucket policy restricting access to your
  CI role.

## Vs Already Cataloged

- **Vs [`bacon`](../bacon/):** orthogonal — `bacon` is a Rust
  watch-and-rebuild loop on top of `cargo`. Pair them: `bacon`
  decides *when* to rebuild, `sccache` makes that rebuild
  cheap. (`RUSTC_WRAPPER=sccache bacon`.)
- **Vs [`turso-cli`](../turso-cli/) / [`mise`](../mise/):**
  unrelated, just shared the "developer-tooling" shelf.
- **Vs `ccache` (not in the catalog):** `sccache` is the
  superset for cross-language and shared-backend use; `ccache`
  remains a strong choice for single-machine C/C++ where
  simplicity beats remote backends.

## Caveats

- **The first build is *slower* than no cache.** You pay the
  hash-and-upload cost on every miss. Net win only kicks in
  once the hit rate climbs (typically after 1–2 CI runs on a
  given branch).
- **Object-store latency dominates on a small cache.** Redis on
  the same VPC is sub-ms; S3 is ~50 ms per object. For a 10 000
  object cargo build, that's a meaningful tail latency.
- **Distributed compilation (`sccache-dist`) is a separate
  service** with its own scheduler / build-server processes.
  Most teams stop at the shared-cache feature and never deploy
  `dist`.
- **Compiler flag changes invalidate the cache.** Even
  semantically irrelevant flag reorderings can cause misses
  because the cache key is over the canonicalised flag string.
- **Watch the cache size.** A 50 GB local cache on a 256 GB
  laptop SSD will wreck other things. Set `SCCACHE_CACHE_SIZE`.
