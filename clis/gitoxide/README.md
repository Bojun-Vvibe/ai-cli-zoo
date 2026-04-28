# gitoxide

- **Repo:** https://github.com/GitoxideLabs/gitoxide
- **Version:** v0.53.0 (latest tagged release, 2025)
- **License:** dual-licensed Apache-2.0 OR MIT
  ([LICENSE-APACHE](https://github.com/GitoxideLabs/gitoxide/blob/main/LICENSE-APACHE)
  · [LICENSE-MIT](https://github.com/GitoxideLabs/gitoxide/blob/main/LICENSE-MIT))
- **Language:** Rust
- **Install:** `cargo install gitoxide` · `brew install gitoxide` · prebuilt
  binaries on the GitHub release page · binary names are `gix` (the
  user-facing porcelain) and `ein` (the experimental higher-level tool)

## What it does

`gitoxide` is a from-scratch reimplementation of git in pure safe Rust,
exposing both a library (`gix`) and CLI binaries.

- `gix` provides plumbing-level commands: `gix clone`, `gix fetch`,
  `gix status`, `gix log`, `gix blame`, `gix pack verify`, `gix odb
  stats`, etc. — many of which are dramatically faster than the
  reference C implementation on large repos.
- Object database operations (pack indexing, delta resolution, loose
  object iteration) are multi-threaded by default and stream to memory
  efficiently, making it the fastest open implementation for whole-repo
  scans (`gix odb stats`, `gix repo verify`).
- Network protocol (smart HTTP, ssh, git://) is implemented natively,
  with progress reporting that's structured (JSON-able) instead of
  ANSI-only.
- Designed as a library first: every feature is also a `gix-*` crate
  you can embed (`gix-pack`, `gix-diff`, `gix-blame`, `gix-revwalk`),
  which is why tools like [`onefetch`](../onefetch/) and many Rust
  forge tools depend on it.
- Memory-safe and free of the long tail of C-side CVEs that have
  affected reference git over the years.

## When to use it

Reach for `gix` when you're scripting against very large repositories
and the reference git is the bottleneck — pack verification, blame on
huge histories, full-repo object walks, or fetch on a slow link where
structured progress matters. It's also the right pick when you're
embedding git into another tool: depend on `gix-*` crates instead of
shelling out to `git` and parsing porcelain. As an end-user CLI it's a
fine read-only complement to git — `gix log`, `gix blame`, `gix status`
all work — and it's a useful sanity check when a repo seems corrupt
(`gix repo verify` is more thorough than `git fsck` on packs).

## When NOT to use it

Don't replace `git` with `gix` for daily write-side work yet — the
project explicitly tracks coverage of the git command surface and many
porcelain commands (rebase, cherry-pick, complex merge strategies,
worktree management) are still in progress or absent. Skip it if you
depend on git hooks, custom merge drivers, GPG signing, or filter-repo
workflows; the reference git ecosystem assumes its own implementation.
Skip it for hosting-side workflows that depend on git's exact wire
behaviour at edge cases (some forges and CI runners have not been
tested against `gix` as the client). When in doubt, use `gix` for
read-only fast paths and reference git for everything that mutates a
remote.
