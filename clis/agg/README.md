# agg

- **Repo:** https://github.com/asciinema/agg
- **Latest version:** v1.7.0 (released 2025-10-22)
- **License:** GPL-3.0 — SPDX `GPL-3.0-only` — [LICENSE](https://github.com/asciinema/agg/blob/main/LICENSE)
- **Category:** terminal recording → animated GIF converter

A **standalone Rust converter that turns asciinema `.cast` recordings
into animated GIFs** (and, with `--format`, into MP4 / WebM / PNG
frame sequences via `ffmpeg`). It is the official successor to the
old Node-based `asciicast2gif`: a single static binary that takes a
`.cast` JSON file (or an `https://asciinema.org/a/...` URL directly)
and emits a frame-accurate, themeable, font-embedded animation
suitable for embedding in a README, a blog post, or a release note
where a real `<video>` element is overkill and an asciinema embed is
unwanted. Because the input is a structured terminal recording (not
a screen capture), the resulting GIF reproduces the original cell
grid pixel-perfectly at any size, picks a typeface from a fontconfig
match (`--font-family "JetBrains Mono"`), and supports the standard
ANSI 256-color + truecolor palette plus a built-in theme set
(`asciinema`, `dracula`, `monokai`, `solarized-dark`, `solarized-light`,
`nord`, `kanagawa-*`).

## Why it's interesting

The CLI-zoo demo workflow ends with "show me what this looks like in
a terminal." The two production paths are an asciinema embed
(requires JavaScript on the host page) or a screen-capture GIF (heavy,
blurry, uncroppable). `agg` is the third option: record once with
`asciinema rec demo.cast`, replay locally to verify, then `agg
demo.cast demo.gif` produces a clean, scalable, README-embeddable
artifact in seconds. Version 1.7.0 added the Kanagawa themes, an
auto-cleanup of failed-conversion outputs, a `--quiet` flag, and
bumped the default font size from 14 to 16 for higher legibility on
HiDPI READMEs.

## Install

```bash
# macOS / Linux via Homebrew
brew install agg

# Cargo (any platform with a Rust toolchain)
cargo install --locked agg

# Prebuilt binary
curl -L -o agg https://github.com/asciinema/agg/releases/download/v1.7.0/agg-x86_64-unknown-linux-gnu
chmod +x agg && sudo mv agg /usr/local/bin/
```

## Sample invocation

```bash
# Record a session
asciinema rec demo.cast

# Convert with a theme + custom font
agg --theme dracula \
    --font-family "JetBrains Mono" \
    --font-size 18 \
    --speed 1.5 \
    demo.cast demo.gif

# Convert directly from a public asciinema URL
agg https://asciinema.org/a/569727 demo.gif
```

## Comparable to

- [`vhs`](../vhs/) — Charm's "GIF-as-code" tool that *scripts* a
  terminal session in a `.tape` DSL and renders to GIF / WebM / MP4;
  use `vhs` when you want the recording itself to be reproducible
  from source, use `agg` when you already have a real `.cast`.
- `asciicast2gif` (deprecated) — the Node-based predecessor `agg`
  was written to replace.
