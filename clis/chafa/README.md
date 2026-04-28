# chafa

- **Repo:** https://github.com/hpjansson/chafa
- **Version:** 1.18.1 (latest stable, February 2026)
- **License:** LGPL-3.0-or-later ([COPYING](https://github.com/hpjansson/chafa/blob/master/COPYING))
- **Language:** C (libchafa) + small CLI wrapper
- **Install:** `brew install chafa` · `pacman -S chafa` · `apt install chafa` · `dnf install chafa` · `nix-env -iA nixpkgs.chafa` · static binaries via release tarball (build from source) · binary name is `chafa`

## What it does

`chafa` renders images, GIFs, and even short video clips as terminal
graphics, picking the **best output mode the host terminal actually
supports**: full-resolution **Sixel** on `xterm` / `mlterm` /
`foot` / `wezterm` / `mintty`, **iTerm2 inline-image protocol** on
iTerm2 / WezTerm / `tmux ≥ 3.5`, **Kitty graphics protocol** on
`kitty` / `ghostty` / `wezterm` / `konsole 22+`, and a
high-quality **Unicode half-block / quarter-block / Braille / ASCII**
fallback when nothing fancier is available — all via the same
`chafa image.png` command, with terminfo / `XTGETTCAP` /
`COLORTERM` / `TERM_PROGRAM` autodetection picking the format. Beyond
the format auto-pick, the differentiator is **`libchafa`** itself: a
GPU-free SIMD-accelerated dithering / colour-quantisation engine that
turns a 4K JPEG into a tasteful 80×24 cells with proper gamma, dithering
(Floyd-Steinberg, Bayer ordered, none), colour-space-aware downscaling,
and a **symbol selector** that lets you trade fidelity for purity (only
ASCII for `--symbols ascii`, only Block Elements for the classic
chunky-pixel look, all of Unicode for "looks like a real photo at 240
columns"). Reads PNG / JPEG / GIF / WebP / TIFF / BMP / QOI / SVG /
AVIF (with the right system codecs), animated GIF / animated WebP with
true frame timing, and pipes from `stdin`; can also enumerate fonts and
**preview every glyph in a font** (`chafa --view-symbol-map`).

## When to pick it / when not to

Pick `chafa` whenever you need an image to appear *in* a terminal —
quick "what does this attachment look like" inside an SSH session,
showing an avatar in a TUI, rendering a chart that another tool just
wrote to PNG, embedding a graph in a Markdown-pager workflow with
[`mdcat`](../mdcat/), or making a `cowsay`-style banner for a shell
prompt out of a logo. It is the right tool when the terminal might be
anything from a Sixel-capable `xterm` over SSH down to `screen`
without colour support — `chafa`'s autodetection handles the
degradation gracefully so you can ship one command in a script and not
worry about who is reading the output. Pair it with
[`viu`](../viu/) when you only need the ANSI-only subset and want a
zero-dep static Rust binary, with [`tv`](../tv/) for tabular previews,
with [`mdcat`](../mdcat/) which already calls Kitty / iTerm2 protocols
for inline images in Markdown but happily delegates to terminal-graphics
fallbacks when it can't, and with [`bat`](../bat/) for the text side of
the same "preview anything in the terminal" flow.

Skip it inside CI logs, plain-text emails, or any terminal you don't
control the codepage of — even the Unicode-fallback output is
copy-paste-hostile and `--format symbols --symbols ascii` is still
huge. Skip it for **video playback** as a primary use — `chafa`
animates GIFs and short MP4-via-`pipeline` clips fine, but for
real video reach for `mpv --vo=sixel` or `mpv --vo=kitty`. Skip it
for **plotting** — it renders pixels, not data; for terminal plots
use `gnuplot -p set terminal sixelgd`, `youplot`, or `plotext`.
Finally, on Apple Terminal.app the situation is "ASCII / 256-colour
fallback only" because Apple still hasn't shipped Sixel, iTerm2, or
Kitty graphics — switch to iTerm2, Ghostty, WezTerm, or Kitty if you
care about quality on macOS.

## Example invocations

```bash
# Just show me the image
chafa cat.jpg

# Constrain to a specific cell size (width x height in cells)
chafa --size 80x24 chart.png
chafa --size 120x cat.jpg          # auto-pick height to preserve aspect

# Force a specific output format
chafa --format sixel poster.png
chafa --format iterm  poster.png
chafa --format kitty  poster.png
chafa --format symbols --symbols ascii  ci-friendly.png
chafa --format symbols --symbols block  classic-chunky.png

# Animate a GIF / WebP at native frame rate
chafa --duration 5 spinner.gif

# Pipe an image from stdin (e.g. fresh from curl)
curl -sSL https://example.com/logo.png | chafa --size 60x

# Preview a whole image directory in your shell
for f in *.png; do printf '\n=== %s ===\n' "$f"; chafa --size 60x20 "$f"; done

# Use a specific palette / dithering combo
chafa --colors 256 --dither bayer chart.png
chafa --colors full --dither none photo.jpg

# Show the symbol set chafa can pick from (debug / fine-tune)
chafa --view-symbol-map block,space,solid

# Render an SVG (requires librsvg)
chafa diagram.svg --size 100x

# Strip ANSI for a non-graphical paste (still useful for Slack code blocks)
chafa --format symbols --symbols ascii --colors 16 logo.png > logo.txt
```
