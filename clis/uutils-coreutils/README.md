# uutils-coreutils

- **Repo:** https://github.com/uutils/coreutils
- **Version:** 0.8.0 (latest stable, 2026-04-06)
- **License:** MIT ([LICENSE](https://github.com/uutils/coreutils/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install uutils-coreutils` · `cargo install coreutils --locked` · `apt install rust-coreutils` (Debian/Ubuntu) · prebuilt binaries on the GitHub release page

## What it does

`uutils-coreutils` is a from-scratch Rust rewrite of GNU coreutils —
the ~100 small Unix utilities (`ls`, `cp`, `mv`, `rm`, `cat`, `cut`,
`sort`, `uniq`, `tr`, `wc`, `head`, `tail`, `sed`-via-`tr`, `dd`,
`du`, `df`, `stat`, `find`-adjacent helpers like `basename` /
`dirname` / `realpath`, plus the textual ones like `printf` / `seq`
/ `yes` / `tac` / `nl` / `expand` / `unexpand` / `fold` / `fmt` /
`paste` / `comm` / `join` / `split` / `csplit` / `od` / `hexdump`)
that every shell pipeline depends on. The project ships them as one
multicall binary `coreutils` (you invoke `coreutils ls -la` or
symlink `ls -> coreutils` and just run `ls`) plus per-utility
binaries for distros that prefer that layout. As of 0.8.0 the
project is the **default coreutils on Ubuntu 25.10 and the planned
default for Ubuntu 26.04 LTS** — meaning this is no longer a
hobbyist replacement, it is becoming the upstream coreutils for a
mainstream LTS distribution.

The compatibility goal is *bug-for-bug* parity with GNU coreutils
on flag semantics and exit codes, validated against the GNU
project's own test suite (the project tracks pass-rate publicly and
is currently >90% across most utilities, with a long tail of edge
cases in `dd` / `sort` / `cp --reflink` / SELinux-attribute
handling). Where uutils intentionally diverges it is documented per
utility — a small set of GNU extensions are gated behind explicit
flags rather than enabled by default. The wire-level flag surface
is otherwise identical, so existing `Makefile` / shell-script /
ansible-playbook invocations port over with zero edits in the
common case.

The reasons to care, beyond the obvious "Rust memory safety in the
TCB of every shell pipeline on the planet": the binaries are
**statically linkable** (one `cargo install --target
x86_64-unknown-linux-musl coreutils` produces a fully static
~5 MB blob you can drop into a `FROM scratch` container as the
entire userland), **cross-platform** (the same code builds on Linux
/ macOS / Windows / WASI — `cp` and `ls` *work* on Windows in a way
GNU coreutils' Cygwin port has never quite achieved), and
**embeddable** (each utility is also a Rust library crate so a
build tool can call `uucore::ls::uumain(...)` directly without
shelling out).

## Pick over / pair with

- **Pick over GNU coreutils** when (a) the deployment target is a
  `FROM scratch` container or a stripped-down embedded image where
  glibc + GNU coreutils is more bytes than the app itself; (b) the
  target is Windows or macOS and you want the *same* coreutils flags
  to work as on Linux without Homebrew's `gcoreutils` prefix dance
  or WSL; (c) the security posture demands "no C in the TCB of the
  init scripts"; (d) the distro is Ubuntu 25.10+ and uutils is
  already the default (don't fight it).
- **Pick over [BusyBox](https://busybox.net) / Toybox** when memory
  safety + standard-flag-compat with GNU matters more than absolute
  minimum binary size. BusyBox is ~1 MB stripped and intentionally
  uses a non-GNU flag dialect for size; uutils is ~5-15 MB and
  matches GNU flags. Embedded routers want BusyBox; container base
  images want uutils.
- **Pick over reaching for individual rewrites** ([`fd`](../fd/),
  [`ripgrep`](../ripgrep/), [`bat`](../bat/), [`eza`](../eza/),
  [`sd`](../sd/), [`dust`](../dust/), [`procs`](../procs/),
  [`bottom`](../bottom/), [`bandwhich`](../bandwhich/)) when the
  goal is "replace the *base* `ls` / `cp` / `mv` so every script
  benefits" rather than "give the human a nicer interactive tool".
  Those tools are *better* than uutils for human interactive use
  (color, fuzzy paths, friendlier output) — keep both: uutils as the
  scriptable base, fd/rg/bat/eza/sd as the human-facing layer.
- **Pair with [`apko`](../apko/)** to assemble a `FROM scratch`
  container image whose entire userland is one statically linked
  uutils binary plus the application; the resulting image is
  routinely 10-30 MB end-to-end with no glibc, no busybox, no
  `bash`-isms in the init scripts.
- **Pair with [`syft`](../syft/) / [`grype`](../grype/)** for
  supply-chain scanning of the resulting minimal image — the
  vulnerability surface drops dramatically when half the CVE feed
  no longer applies because there is no glibc / coreutils-in-C in
  the image.
- **Pair with [`watchexec`](../watchexec/) / [`entr`](../entr/)** for
  the script-driven-reload loop where every shell utility in the
  pipeline is part of the trust boundary — uutils removes the C
  parts of that boundary.

## Caveats

- Compatibility is *very high* but **not 100%** with GNU coreutils.
  The known gaps are: `dd` (some `conv=` modes), `sort` (locale-aware
  collation under non-trivial `LC_COLLATE`), `cp --reflink=always`
  on filesystems where the syscall semantics differ subtly,
  SELinux/xattr/ACL preservation flags on some utilities, and a
  handful of GNU extensions that are intentionally not implemented.
  Read the per-utility status table on the upstream README before
  switching a production-critical script.
- The multicall binary mode means symlinks are required to expose
  individual utility names; distros usually do this for you. If you
  install via `cargo`, you get one `coreutils` binary and need to
  symlink (or just invoke as `coreutils <util>`).
- Switching the *default* `/usr/bin/ls` etc. on a non-Ubuntu-25.10
  distro is real operational risk — install side-by-side first
  (e.g. under `/usr/local/bin/uu-*`), measure for a release cycle,
  then flip the symlinks.
- Performance is generally *competitive* with or *better than* GNU
  coreutils on Linux for the common cases (`ls`, `cat`, `cp`, `wc`)
  and on par or slightly slower for the I/O-syscall-bound utilities
  (`dd`, large `sort` on disk-spilling workloads). Run the GNU test
  suite on your hot paths if performance matters.
- License is **MIT** for the project itself; some embedded test
  fixtures inherit GNU coreutils' GPLv3-or-later test data and are
  segregated under the `tests/` tree — the shipped binaries are
  MIT-licensed.

## Why it's in this catalog

Coding agents and CI systems both lean enormously on shell utilities
in their inner loops — `find | xargs sort | uniq -c | head` is the
agent's grep-replacement, the CI's log-summariser, and the deploy
script's safety check, all in one pipeline. Replacing the C-implemented
TCB underneath those pipelines with a Rust implementation that can
also produce a `FROM scratch` container userland is the kind of
"every workflow gets a little safer" change that is worth knowing
about even if the day-1 invocation looks identical.
