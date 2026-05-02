# bingrep

> **Greps through binaries from various OSs and architectures, and
> colors them.**
> A Rust CLI that parses ELF / Mach-O / PE / archive / Java class
> files and prints a colored, structured dump of their headers,
> sections, symbols, relocations, and dynamic entries.
> Pinned to **v0.12.1**
> ([LICENSE](https://github.com/m4b/bingrep/blob/master/LICENSE),
> MIT).

Source: <https://github.com/m4b/bingrep>

## TL;DR

`bingrep` is a single-binary, no-config inspector for executable
and object files. Hand it a path and it figures out the format
(ELF on Linux, Mach-O on macOS, PE on Windows, `ar` archives,
Java `.class`), then dumps the parsed structure with ANSI colors
applied to byte ranges, addresses, symbols, and section flags.
It is built on top of `goblin`, the same Rust parser used by
several reverse-engineering toolchains, so the parse is
trustworthy on stripped, packed, or cross-compiled binaries
where `readelf` / `otool` sometimes give terse or
platform-specific output.

## Install

```bash
# Cargo (the canonical install path)
cargo install bingrep

# Homebrew (macOS / Linuxbrew)
brew install bingrep

# Arch
pacman -S bingrep   # or via AUR depending on the repo state

# from source
git clone https://github.com/m4b/bingrep && cd bingrep
cargo build --release
install -m 0755 target/release/bingrep /usr/local/bin/

# verify
bingrep --version       # bingrep 0.12.1
```

## License

MIT — see
[LICENSE](https://github.com/m4b/bingrep/blob/master/LICENSE).
Permissive, embed-friendly. Safe to ship inside an internal
binary-triage tool or a CI image; no copyleft, attribution
limited to preserving the notice. The `goblin` parser
dependency is also MIT.

## One Concrete Example

```bash
# 1. Inspect a Linux ELF the same way readelf -a would, but colored
bingrep /usr/bin/ls
# emits: file header (class / endianness / machine / entry),
# program headers, section headers, dynamic table, symbols,
# relocations — each block colored so you can scan for the one
# field you care about.

# 2. Inspect a macOS Mach-O (cross-platform parse)
bingrep /bin/ls
# Mach-O header, load commands (LC_SEGMENT_64, LC_DYLD_INFO,
# LC_LOAD_DYLIB), sections per segment, symbol stubs — all from
# the same binary that just parsed an ELF.

# 3. Inspect a Windows PE from a Linux box (no wine, no objdump)
bingrep ./malware-sample.exe
# COFF header, optional header, data directories, section table,
# imports / exports — useful in CI when you cross-compile to
# Windows and want to verify the import table from a Linux runner.

# 4. Inspect every member of a static archive in one pass
bingrep libfoo.a
# walks the `ar` index, parses each member object, and prints
# the per-object dump back-to-back — saves the ten-step
# `ar t` / `ar x` / `readelf -a foo.o` loop.

# 5. Search for a string across symbols (the "grep" in bingrep)
bingrep --search _start /usr/bin/ls
# parses the symbol table and filters to entries containing the
# substring; faster than `nm | grep` because it reuses the
# parsed structure.

# 6. Inspect a Java .class file (header, constant pool, methods)
bingrep target/classes/Foo.class
# JVM magic, version, constant pool with resolved string refs,
# field / method tables — without booting `javap`.
```

## Niche It Fills

**The "what is actually in this binary, on any host, in one
command" gap.** The platform-native tools (`readelf` / `objdump`
on Linux, `otool` / `nm` on macOS, `dumpbin` on Windows) each
parse exactly one format and print platform-flavoured output.
When you have a directory of cross-compiled artifacts — `.so`,
`.dylib`, `.dll`, `.a`, `.exe`, stripped, signed, statically
linked — running the right tool for each one is friction, and
the output formats don't compose. `bingrep` is the *one* binary
that parses all of them with consistent colored output, so
"diff the imports of the Linux build against the macOS build"
or "spot the missing symbol in this cross-compile" becomes a
shell loop instead of a per-platform investigation.

## Why use it

Three things `bingrep` does that the obvious alternatives don't:

1. **Cross-format parse from one binary.** `readelf` parses ELF,
   `otool` parses Mach-O, `dumpbin` parses PE — each is
   unavailable on the other two OSes without emulation.
   `bingrep` parses all three (plus `ar` and Java class) on any
   host, so a Linux CI runner can introspect a Windows artifact
   without installing wine or a Windows-native toolchain.
2. **Colored output is the default, not an afterthought.**
   Section flags, symbol bindings, relocation types, and
   addresses each get their own color class. Scanning a 5000-
   symbol dump for the one weak symbol that broke the link is
   eyes-on-color, not pipe-into-grep.
3. **`goblin` parser pedigree.** `goblin` is a battle-tested
   pure-Rust binary parser used by `cargo-binutils`, several
   exploit-mitigation auditors, and reversing toolkits. It
   handles malformed / packed / non-canonical files more
   robustly than the `binutils` family, which sometimes refuses
   on truncated or non-conformant inputs.

For an LLM-CLI workflow, `bingrep` is the **binary-evidence
step**: an agent claims "the released artifact does not export
`init_foo`", a CI step runs `bingrep -e ./libfoo.so | grep
init_foo` and either confirms or refutes against the actual
parsed export table. It's the structural counterpart to
[`hexyl`](../hexyl/) — `hexyl` shows you the bytes,
`bingrep` shows you what those bytes *mean* per the format spec.

## Vs Already Cataloged

- **Vs [`hexyl`](../hexyl/):** `hexyl` is a colored hex dumper
  that knows nothing about file structure — it shows offset, hex
  bytes, ASCII gutter, byte-class colouring. `bingrep` parses
  the file format and tells you "those bytes at 0x40 are the
  ELF program-header offset, value 0x2A8". Use `hexyl` to look
  at raw bytes; use `bingrep` to look at parsed structure.
- **Vs `readelf` / `objdump`:** GNU `binutils` are the entrenched
  answer on Linux, but ELF-only and verbose. `bingrep` is
  cross-format and colored; `objdump -d` is still the right
  answer when you need actual *disassembly* of the `.text`
  section, which `bingrep` does not do.
- **Vs `nm` / `strings`:** Both extract one slice (symbols /
  printable runs) without showing structure. `bingrep` shows the
  symbol *table* in context (which section, which binding, which
  type) instead of a flat list.
- **Vs `radare2` / `rizin` / `ghidra`:** Those are full reverse-
  engineering platforms — disassembly, decompilation, scripting,
  graphs. `bingrep` is the ten-second triage view *before* you
  decide it's worth opening a real RE tool. Different intensity.

## Caveats

- **Inspector, not disassembler.** `bingrep` does not show
  instruction-level disassembly of `.text`. If you need
  "what does function X do", reach for `objdump -d`,
  `radare2`, or a decompiler. `bingrep` stops at the format
  layer.
- **No DWARF / debug-info pretty-printing.** It will tell you
  the `.debug_info` section exists and how big it is, not the
  resolved type tree. For DWARF parsing reach for `addr2line`,
  `llvm-dwarfdump`, or `dwarfsplit`.
- **Output is for humans.** No `--format=json` flag at v0.12.1,
  so machine consumption means parsing the colored text (or
  forking the underlying `goblin` library directly from Rust).
- **Coverage is "the obvious formats".** ELF / Mach-O / PE /
  `ar` / Java `.class` are well-supported; OCaml `.cmo`, .NET
  metadata streams, WebAssembly modules, COFF object variants,
  XCOFF, and packed / obfuscated binaries are out of scope or
  best-effort. For Wasm reach for `wasm-tools`.
- **Tag cadence is slow.** v0.12.1 (2024) is the latest release;
  the project is feature-complete relative to its goals and
  rides on `goblin` for parser fixes. Don't expect rapid new
  feature releases.
