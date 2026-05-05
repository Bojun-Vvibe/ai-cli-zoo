# findutils (uutils)

> **A Rust rewrite of GNU findutils — `find`, `xargs`, `locate`,
> `updatedb` — that aims for byte-compatible CLI surface and
> drop-in replacement on Linux / macOS / Windows.** Sister project
> to [`uutils-coreutils`](../uutils-coreutils/), same MIT licence,
> same single-binary-multiplexer philosophy. Pinned to **0.8.0**
> ([LICENSE](https://github.com/uutils/findutils/blob/main/LICENSE),
> MIT).

Source: <https://github.com/uutils/findutils>

## TL;DR

`find` is the canonical "walk the filesystem and act on what matches"
verb in the Unix toolbox, and the modern catalog has many *new-shape*
alternatives ([`fd`](../fd/), [`fselect`](../fselect/),
[`erdtree`](../erdtree/)) — but no *drop-in compatible* one. `uutils
findutils` fills exactly that gap: the same flag set as GNU
(`-name`, `-iname`, `-type`, `-mtime`, `-perm`, `-exec … {} \;`,
`-print0`), the same exit codes, the same predicate composition
(`( … -o … ) -prune`), shipped as one MIT-licensed Rust binary so
distributions, embedded images, and Windows toolchains can replace
the GPL-licensed GNU implementation without rewriting any script.

## Install

```bash
# Cargo (installs find / xargs / locate / updatedb)
cargo install findutils

# Pre-built binaries:
# https://github.com/uutils/findutils/releases/tag/0.8.0

# Verify
find --version    # find (uutils) 0.8.0
```

The Rust binaries install under their canonical names. To run them
side-by-side with GNU findutils, prefix `PATH` selectively or invoke
via `cargo run --bin find -- …` from a checkout.

## License

MIT — see
[LICENSE](https://github.com/uutils/findutils/blob/main/LICENSE).
Identical licence story to
[`uutils-coreutils`](../uutils-coreutils/): permissive replacement
for the GPL GNU originals.

## One Concrete Example

```bash
# 1. classic recursive name search (identical syntax to GNU find)
find . -type f -name '*.rs' -print

# 2. predicate composition + null-delimited output for safe xargs
find . -type f \( -name '*.log' -o -name '*.tmp' \) -print0 \
  | xargs -0 rm -v

# 3. mtime-window cleanup (older than 30 days, larger than 100 MB)
find /var/cache -type f -mtime +30 -size +100M -delete

# 4. -exec with batch (-+) form
find . -name '*.png' -exec optipng -quiet {} +

# 5. -prune to skip a subtree
find . -path ./node_modules -prune -o -type f -name '*.ts' -print
```

## Niche It Fills

**A permissively-licensed, cross-platform `find` whose CLI does not
diverge from GNU.** The catalog has plenty of *better-shaped* file
finders:

- [`fd`](../fd/) — fast, ergonomic, regex-default — but a *new* CLI
  shape (`fd -e rs` not `find . -name '*.rs'`).
- [`fselect`](../fselect/) — SQL-shaped (`SELECT name FROM .`).
- [`erdtree`](../erdtree/) — tree-rendered finder.
- [`tre`](../tre/) — git-aware tree walker.

`uutils findutils` exists for the *opposite* requirement: the
existing 200-line shell script, embedded build system, or Yocto
recipe that already encodes thousands of `find` invocations and
*cannot be rewritten*, but where shipping GNU findutils is
inconvenient (licence audit, Windows portability, container size,
musl-static cross-compile).

## Why use it

1. **Drop-in CLI compatibility is the product.** Same flags, same
   composition, same exit codes — every existing `find` script keeps
   working. The new-shape alternatives ([`fd`](../fd/) etc.) deliver
   a better tool by *breaking* compatibility; this one delivers a
   replacement by *keeping* it.
2. **MIT licence relaxes redistribution.** Replace the GPL GNU
   binaries in commercial firmware, container base images, and
   vendored toolchains without the MIT-vs-GPL audit conversation
   the original triggers.
3. **Cross-platform single binary.** Native Windows builds without
   Cygwin / MSYS2; static-musl Linux builds for distroless containers;
   universal macOS binaries. The original needs a POSIX environment.
4. **Shared maintenance with [`uutils-coreutils`](../uutils-coreutils/).**
   Same project, same Rust crate ecosystem, same test methodology
   (BFS-style compatibility tests against GNU output).

## Vs Already Cataloged

- **Vs [`fd`](../fd/):** orthogonal, not competitive — `fd` is a
  *better* file finder for human use (regex by default, `.gitignore`
  honouring, parallel walk, coloured output), but its CLI does not
  match `find`. Pick `fd` for new scripts and interactive use; pick
  `uutils findutils` when you need to replace `find` in an existing
  pipeline without rewriting it.
- **Vs [`fselect`](../fselect/):** different paradigm — SQL queries
  over the filesystem. Powerful for ad-hoc analytics, not a drop-in
  for `-exec`-style workflows.
- **Vs [`uutils-coreutils`](../uutils-coreutils/):** sibling project
  in the same uutils umbrella. coreutils covers `ls` / `cp` / `mv` /
  `cat` / `wc` / `sort` / etc.; findutils covers `find` / `xargs` /
  `locate` / `updatedb`. Install both for a complete MIT-licensed
  Unix base.
- **Vs GNU findutils (the upstream this targets):** identical CLI,
  MIT vs GPL-3.0, Rust vs C, single static binary vs distro
  packaging. Compatibility is intentionally near-100%; subtle
  divergences are tracked as bugs in the uutils issue tracker.

## Caveats

- **Not yet 100% feature parity.** A handful of obscure GNU
  predicates (`-context` for SELinux, some `-newerXY` combinations,
  `locate`'s mlocate-database compatibility) are still in progress.
  Audit your script's predicate set against the project's BFS
  compatibility matrix before swapping in production.
- **Performance is comparable, not categorically better.** This is
  a *compatibility* project — for raw speed on large trees,
  [`fd`](../fd/) is the right pick (parallelism, smarter
  traversal).
- **`locate` / `updatedb` need an index.** As with the GNU original,
  `locate` reads a database that `updatedb` builds. The project's
  database format aims to be compatible with mlocate; verify against
  your existing `/var/lib/mlocate/mlocate.db` before relying on
  it.
- **`-exec command \;` shell-quoting matches GNU.** Includes the
  backslash-semicolon terminator quirk; review the project's
  compatibility notes if you use exotic predicate combinations.
