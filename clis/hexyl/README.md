# hexyl

> **Colourful command-line hex viewer** — a single Rust binary by
> the author of `bat` / `fd` / `hyperfine` that pretty-prints the
> bytes of a file as a three-pane hexdump (offset · hex · ASCII)
> with byte-class colouring (NULs, printable ASCII, ASCII
> whitespace, low-control, high-control, non-ASCII), Unicode box
> drawing for the panel borders, and a paged-output default that
> hands off to `less -RFX`. Pinned to **v0.17.0** (commit
> `8eb6d4771ce1ec7af65d06bd335457783b77d557`,
> [LICENSE-MIT](https://github.com/sharkdp/hexyl/blob/master/LICENSE-MIT)
> + [LICENSE-APACHE](https://github.com/sharkdp/hexyl/blob/master/LICENSE-APACHE),
> dual MIT / Apache-2.0).

Source: <https://github.com/sharkdp/hexyl>

## TL;DR

`hexyl` is what you reach for when `xxd file.bin | less` stops
being enough — when you need to *see at a glance* where the NUL
padding ends, where the printable header sits, where the
high-bit binary payload begins, and what byte boundaries the
records align on. Each byte category gets its own colour, so a
PNG signature, a TLV record, or a corrupted UTF-8 stretch is
visible without counting columns. It is a strict subset of `xxd`
in features (no patch-shaped output, no reverse mode), but the
defaults are tuned for *reading* binary data as a human, not for
generating C `unsigned char[]` literals.

## Install

```bash
# Homebrew (macOS / Linux)
brew install hexyl

# Cargo
cargo install --locked hexyl

# Linux package managers
# Arch:    pacman -S hexyl
# Debian/Ubuntu (24.04+): apt install hexyl
# Fedora:  dnf install hexyl
# Nix:     nix-env -iA nixpkgs.hexyl
# Alpine:  apk add hexyl

# from a release tarball (any OS)
curl -Lo hexyl.tar.gz "https://github.com/sharkdp/hexyl/releases/download/v0.17.0/hexyl-v0.17.0-aarch64-apple-darwin.tar.gz"
tar xf hexyl.tar.gz --strip-components=1
sudo install hexyl /usr/local/bin/

# verify
hexyl --version    # hexyl 0.17.0
```

`hexyl` honours `NO_COLOR` and `--color={always,auto,never}`,
detects whether stdout is a TTY for paging, and respects `PAGER`
(falls back to `less -RFX`).

## License

Dual-licensed MIT or Apache-2.0 — see
[LICENSE-MIT](https://github.com/sharkdp/hexyl/blob/master/LICENSE-MIT)
and
[LICENSE-APACHE](https://github.com/sharkdp/hexyl/blob/master/LICENSE-APACHE).
Pick whichever your project prefers.

## One Concrete Example

```bash
# 1. view the first 64 bytes of an unknown binary
hexyl --length 64 mystery.bin

# 2. inspect a PNG header — magic bytes + IHDR chunk
hexyl --length 32 image.png
# 0x89 0x50 0x4e 0x47 0x0d 0x0a 0x1a 0x0a   ‰PNG....
# 0x00 0x00 0x00 0x0d 0x49 0x48 0x44 0x52   ....IHDR
# ...

# 3. seek into the middle of a large file
hexyl --skip 0x1000 --length 256 firmware.img

# 4. pipe arbitrary bytes through it
curl -s https://example.com/ | hexyl --length 128

# 5. tighter screen layout (1 panel, 16 bytes wide)
hexyl --panels 1 --bytes-per-line 16 small.bin

# 6. find a marker inside a binary, view the surrounding region
offset=$(grep --byte-offset --only-matching --text 'BEGIN_PAYLOAD' big.bin | head -1 | cut -d: -f1)
hexyl --skip "$offset" --length 512 big.bin

# 7. compare two builds byte-for-byte (with diff-friendly output)
hexyl --plain a.bin > a.hex; hexyl --plain b.bin > b.hex; diff -u a.hex b.hex
```

## Niche It Fills

**Visual hex inspection at a TTY, not a hexdump generator.**
`xxd`, `od`, and `hexdump` exist to *produce* hex output suitable
for embedding in scripts or patching back with `xxd -r`. `hexyl`
exists to make a human stare at a binary file for ten seconds
and *see* its structure — colour categorises bytes faster than
the eye can parse hex digits, and the box-drawn panels keep
offset / hex / ASCII columns distinct under terminal wrapping.

## Why use it

Three things `hexyl` does that `xxd` / `od` / `hexdump` do not,
that pay back the install:

1. **Byte-class colouring.** NULs (often padding) are dim, ASCII
   printable is one colour, ASCII whitespace another, low control
   bytes a third, high-bit non-ASCII a fourth. A header / body /
   trailer split in a binary becomes visible as colour bands
   without you reading the values.
2. **Defaults tuned for reading.** Two-panel layout, `0x` prefix
   on offsets, Unicode borders between offset / hex / ASCII, and
   automatic paging through `less -RFX` when output is long.
   `xxd` requires you to assemble the equivalent ergonomics with
   flags every time.
3. **`--skip` / `--length` accept hex, decimal, and SI/IEC
   suffixes.** `hexyl --skip 0x1000 --length 4KiB` reads exactly
   the page you want; `dd skip=... bs=... | xxd` is the
   equivalent and is one extra moving part with footgun-shaped
   defaults.

## Vs Already Cataloged

- **Vs [`bat`](../bat/):** orthogonal — `bat` is a text-file
  pretty-printer (syntax highlighting, line numbers, paging) for
  source code; `hexyl` is the same author's binary-file
  pretty-printer. Pair them: `bat` for `.rs` / `.json`, `hexyl`
  for `.bin` / `.png` / `.tar.zst`.
- **Vs [`fd`](../fd/) / [`ripgrep`](../ripgrep/):** orthogonal —
  those are *find* tools (locate files / lines); `hexyl` is a
  *display* tool for the bytes inside a single file you already
  found.
- **Vs `xxd` (not cataloged):** `xxd` ships with Vim and is
  everywhere, supports `-r` reverse-patching, and is the right
  answer for "I need to embed bytes in a C source file" or
  "patch a binary with `xxd -r`." `hexyl` is the right answer
  everywhere else: better colours, better defaults, better
  paging, faster visual structure detection. Keep `xxd` for
  scripted patching; reach for `hexyl` for reading.
- **Vs `od` / `hexdump` (not cataloged):** historical tools with
  full POSIX format-string control over the output shape.
  Indispensable when you need a specific column layout for a
  downstream parser. Useless for "what does this `.bin` look
  like?" — `hexyl` wins by an order of magnitude on that task.

## Caveats

- **Read-only.** `hexyl` does not patch bytes, has no `-r`
  reverse mode, and does not write back to the input. For
  surgical edits use a real hex editor (`hexedit`, `wxHexEditor`,
  `radare2 -w`).
- **No structure decoding.** `hexyl` shows raw bytes; it does
  not know about PE / ELF / DWARF / TLV / protobuf shapes. Pair
  with `binwalk`, `readelf`, `objdump`, or a `protoc --decode_raw`
  pipeline when you need semantic decoding.
- **Colour is the product.** On a non-TTY pipe colour is off by
  default; on dumb terminals or `NO_COLOR` environments most of
  the value of `hexyl` over `xxd` evaporates. Use `--color=always`
  if you are piping through `less -R` yourself.
- **Large files: bring `--length`.** `hexyl massive.iso` will
  happily try to render the whole thing through the pager. Always
  bound it with `--length` (and `--skip` to seek) or pipe through
  `head -c` first.
