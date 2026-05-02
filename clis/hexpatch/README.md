# hexpatch

> **A binary patcher and hex editor with a terminal user
> interface** — a single Rust TUI that opens an arbitrary file
> as hex, decodes it as ELF / Mach-O / PE when it recognizes
> the magic, shows the disassembly side-by-side with the raw
> bytes, and lets you patch instructions in place using
> Keystone (the assembler from the Capstone family). Pinned to
> **v1.12.6**
> ([LICENSE](https://github.com/Etto48/HexPatch/blob/master/LICENSE),
> MIT).

Source: <https://github.com/Etto48/HexPatch>

## TL;DR

`hexpatch` is the rare hex editor that knows what it's
looking at. Open a binary and it splits the screen into a
hex view and a Capstone-disassembled instruction view; jump
to any instruction and you can rewrite it as assembly
(`mov rax, 0`) and the tool re-assembles to bytes via
Keystone, NOPs / pads correctly, and writes back in place
without shifting offsets — the thing GUI hex editors like
`hexedit`/`bvi`/`heh` make you do by hand. It speaks
x86 / x86-64 / aarch64 / arm out of the box, parses ELF /
Mach-O / PE headers so you can navigate to a section /
symbol by name instead of byte offset, and ships an
**SSH/SFTP** mode so you can patch a binary on a remote box
without scping it back and forth (handy for CTF boxes,
embedded targets, or "the bug only repros on staging"). Lua
scripting lets you extend keybindings and write small
analysis helpers. Not a replacement for Ghidra / radare2 /
Binary Ninja for analysis — it's a focused **edit + save**
tool for when you already know which instruction you want
to change.

## Install

```bash
# cargo (Rust toolchain, always latest)
cargo install hex-patch

# Single-binary download (GitHub releases, ~24 MB)
curl -L -o hex-patch \
  https://github.com/Etto48/HexPatch/releases/download/v1.12.6/hex-patch-macos-latest
chmod +x hex-patch && sudo mv hex-patch /usr/local/bin/

# Build from source (Rust >= 1.74)
git clone --depth 1 --branch v1.12.6 \
  https://github.com/Etto48/HexPatch.git
cd HexPatch && cargo build --release
```

## Usage

```bash
# Open a local binary; press Tab to toggle hex<->asm pane
hex-patch ./target/release/myapp

# Patch a function prologue: navigate to it, hit `p`, type
#   xor rax, rax
#   ret
# Save with Ctrl+S — bytes are re-assembled in place.

# Edit a binary on a remote host over SSH/SFTP without copying it
hex-patch ssh://user@host:/usr/local/bin/myservice
```

## Why it's interesting

The TUI hex-editor space is crowded with read-mostly tools
(`xxd`, `hexyl`, `bingrep`) and old-school editors (`bvi`,
`hexedit`, `heh`) that treat a binary as an undifferentiated
byte stream. `hexpatch` is the only actively-maintained one
that combines (a) Keystone re-assembly so you can edit at
the instruction level, (b) Mach-O / ELF / PE parsing so you
can jump by symbol, and (c) remote editing over SSH built
in. It fills the same niche as Ghidra's "Patch Instruction"
dialog or Binary Ninja's "Make Code", but as a 24 MB CLI
that runs anywhere — the right tool for a quick "flip this
jne to je and re-test" without firing up a 1.5 GB IDE. Not
the right tool for symbol resolution / decompilation
(reach for radare2 / Ghidra) or for editing huge files like
disk images (the in-memory model isn't streamed).
