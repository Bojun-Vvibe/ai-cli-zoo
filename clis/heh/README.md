# heh

> **A terminal hex viewer / editor with vim-like keybindings**
> — opens a binary file in a side-by-side hex + ASCII view,
> lets you navigate with `hjkl`, jump with `g`/`G`, search,
> and edit bytes in place from a single ~2 MB Rust binary.
> Pinned to **v0.6.0**
> ([LICENSE](https://github.com/ndd7xv/heh/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ndd7xv/heh>

## TL;DR

`heh` (the **H**ex **E**ditor **H**elper) is what you reach for
when `hexyl` is too read-only and `bvi`/`hexedit` feel like 1995.
The interface is the classic three-pane layout — address column
on the left, hex bytes in the middle, ASCII rendering on the
right — but every interaction is modeled on vim: `h/j/k/l` to
move, `i` to enter edit mode, `Esc` to leave it, `:w` to save,
`:q` to quit, `/` to search for a hex pattern or ASCII string.
Crucially, edits are byte-level overwrites (the file size never
changes), which matches how you actually fix things in firmware
images, save files, or binary protocol captures.

It speaks the same "show me 16 bytes per row, color-code the
ASCII" language as `hexyl`, but adds writability, a label bar
showing the current offset in decimal/hex/octal, an endianness
toggle for the integer interpretation popup, and a jump-to-offset
prompt. There is no scripting layer and no hex-grammar diffing —
this is the interactive editor, paired with `hexyl` for piping
and `radare2`/`rizin` for heavyweight reverse engineering.

## Install

```bash
# Cargo (recommended)
cargo install heh

# Homebrew
brew install heh

# Arch Linux
pacman -S heh
```

## Typical usage

```bash
# Open a binary for editing
heh ./firmware.bin

# Inside heh:
#   h j k l       move cursor one byte / row
#   w b           jump word forward / back
#   g  / G        top of file / bottom of file
#   :1234<CR>     jump to decimal offset 1234
#   :0x1F00<CR>   jump to hex offset 0x1F00
#   i             enter edit mode (type hex digits to overwrite)
#   Esc           leave edit mode
#   /DEADBEEF<CR> search for byte sequence DE AD BE EF
#   :w            save in place
#   :q            quit (refuses if unsaved; :q! to force)
```

## Why pick `heh`

- **Vim muscle memory.** If `hjkl`, `:w`, `:q`, and `/` are
  reflex, you are already productive in `heh` with no manual.
- **In-place edits, fixed file size.** Matches what you want for
  patching binaries — no accidental insertions that desync every
  later offset.
- **Tiny and static.** Single Rust binary, no curses build
  weirdness, runs identically on macOS, Linux, and BSDs.
- **Pairs well with `hexyl`.** Use `hexyl` in pipelines (it is
  read-only and dumps), use `heh` when you need to actually
  change a byte and save.
