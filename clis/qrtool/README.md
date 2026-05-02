# qrtool

> **A small, single-binary QR code encoder / decoder for the
> terminal** — generates QR codes from stdin or arguments and
> decodes them back from PNG/SVG/JPEG, all from one Rust binary
> with no Java, no browser, and no `libqrencode` C dependency.
> Pinned to **v0.11.6**
> ([LICENSE-MIT](https://github.com/sorairolake/qrtool/blob/develop/LICENSES/MIT.txt),
> MIT OR Apache-2.0).

Source: <https://github.com/sorairolake/qrtool>

## TL;DR

`qrtool` is the "I just need a QR code in the terminal" tool.
Two subcommands (`encode` and `decode`) cover ~95% of QR
workflows: encode a Wi-Fi SSID, a vCard, a one-time URL, or a
Bitcoin address, and pipe the result straight into a terminal
viewer (it can output as Unicode block art, PNG, SVG, JPEG, BMP,
PIC, ANSI, or plain ASCII), or decode a screenshot you got over
chat to recover the underlying URL or text payload. Encoding
options expose every knob the QR spec defines — error-correction
level (`-l L|M|Q|H`), version (`-v 1..40`), module size,
foreground/background colors, margin, and ECI assignment for
non-UTF-8 byte modes — but the defaults are sane enough that
`qrtool encode "$URL"` produces a scannable code on the first
try in any modern terminal that can render half-blocks.

Decoding leans on the `rxing` Rust port of ZXing, so it handles
rotated, blurred, and partially-occluded images far better than
naive `zbar` wrappers.

## Install

```bash
# Homebrew
brew install qrtool

# Cargo
cargo install qrtool

# Nix
nix-env -iA nixpkgs.qrtool
```

## Typical usage

```bash
# Print a QR for a URL straight to the terminal as Unicode blocks
qrtool encode -t terminal "https://example.com/path"

# Generate a 512x512 PNG with high error correction
qrtool encode -l H -s 8 -o wifi.png \
  "WIFI:T:WPA;S:MyNet;P:hunter2;H:false;;"

# Decode a QR screenshot you received
qrtool decode received.png
# -> https://example.com/path

# Pipe text in, get SVG out
echo "secret payload" | qrtool encode -t svg > payload.svg
```

## Why pick `qrtool`

- **One binary, no runtime deps.** Unlike `qrencode` (C, needs
  PNG libs at build time) or `python-qrcode` (Python + Pillow),
  `qrtool` is a static Rust binary you can drop on a server.
- **Encode *and* decode.** `qrencode` only encodes; `zbarimg`
  only decodes. `qrtool` does both with one tool.
- **Terminal-native output.** `-t terminal` renders directly in
  any UTF-8 terminal — useful for sharing a Wi-Fi password to a
  guest's phone over SSH without ever creating a file.
- **Scriptable.** Exit code is non-zero on decode failure, output
  format is selectable, and stdin/stdout are first-class — drops
  into pipelines cleanly.
