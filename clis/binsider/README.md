# binsider

> **An ELF binary explorer in your terminal** — interactive
> TUI for poking at Linux binaries: ELF header / sections /
> segments, symbol tables (static and dynamic), relocations,
> shared library dependencies, dynamic strings, hex view of
> any region, and a syscall trace tab that runs the binary
> under `strace` and folds the calls into a navigable list —
> all in one keyboard-driven panel layout, with no need to
> remember whether the right tool was `readelf -d`,
> `objdump -T`, `nm -D`, `ldd`, `strings`, or `strace`.
> Pinned to **v0.3.2** (SPDX: `Apache-2.0` / `MIT` dual,
> [LICENSE-APACHE](https://github.com/orhun/binsider/blob/main/LICENSE-APACHE)).

Source: <https://github.com/orhun/binsider>

## TL;DR

`binsider` is a Rust TUI built on `ratatui` that takes one
positional argument — a path to an ELF binary — and opens a
multi-tab inspector: General (file metadata + ELF header),
Static Analysis (sections, segments, symbols, relocations,
notes), Dynamic Analysis (live `strace` / library calls /
runtime strings), Strings (printable strings with filtering),
and Hexdump (any region, with offsets). Tab keys switch
panels, `/` filters the active table, and the whole thing
runs over SSH on a headless box. It is the
"`htop`-for-binaries" tool: live, navigable, pleasant — vs
the half-dozen one-shot text dumpers it replaces.

## Install

```bash
# Cargo
cargo install binsider

# Homebrew
brew install binsider

# Pre-built binary (Linux)
# https://github.com/orhun/binsider/releases/tag/v0.3.2

# verify
binsider --version   # binsider 0.3.2
```

## License

Dual-licensed under Apache-2.0 OR MIT — see
[LICENSE-APACHE](https://github.com/orhun/binsider/blob/main/LICENSE-APACHE)
and
[LICENSE-MIT](https://github.com/orhun/binsider/blob/main/LICENSE-MIT).

## Representative Commands

```bash
# 1. open the TUI on a binary
binsider /usr/bin/ls

# 2. inspect a self-built artefact during development
cargo build --release && binsider target/release/myapp

# 3. inspect a system shared object (sections, exported symbols)
binsider /lib/x86_64-linux-gnu/libssl.so.3
```

Inside the TUI: `Tab` / `Shift-Tab` cycle the top-level tabs
(General, Static, Dynamic, Strings, Hexdump), `j`/`k` move
within a list, `/` filters the active table, `Enter` drills
into the selected row (e.g. jump from a symbol to its
location in Hexdump), and `q` quits.

## Why It Matters

Investigating an ELF binary today means juggling six
single-purpose tools: `file`, `readelf -h`, `readelf -d`,
`objdump -T`, `nm -D`, `ldd`, `strings`, `strace`. Each one
prints a wall of text, none share state, none let you filter
interactively, and the cognitive load of remembering the
right flag for each is the actual bottleneck. `binsider`
unifies these into one TUI with consistent navigation: the
ELF header, sections, segments, dynamic symbols, dependent
libraries, and a live syscall trace are all tabs in the same
window, and the hex view will jump to the offset of any
symbol you select. It's the right tool for reverse
engineering a stripped binary, debugging a "missing symbol
at runtime" link error, auditing what a third-party CLI
actually does on first launch, validating a release build
ships the symbols you expect, or just learning the ELF
format by clicking around it. For Linux-on-the-server work
it is genuinely the fastest way from "what is this binary"
to a useful answer.
