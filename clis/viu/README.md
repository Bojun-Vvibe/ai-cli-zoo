# viu

> **Terminal image viewer with native support for iTerm and Kitty**
> — a single Rust binary that renders PNG / JPEG / GIF / WebP / BMP
> / TIFF / ICO / FARBFELD / animated GIFs directly inside the
> terminal, auto-detecting the iTerm2 inline-image protocol or the
> Kitty graphics protocol when available, and falling back to
> half-block / Sixel rendering elsewhere. Pinned to **v1.6.1**
> (commit `5733be5f8b18c54495123725e7f13cc5ec23ee68`,
> [LICENSE-MIT](https://github.com/atanunq/viu/blob/master/LICENSE-MIT),
> MIT).

Source: <https://github.com/atanunq/viu>

## TL;DR

`viu image.png` prints the image *into* your terminal at the
cursor — pixel-perfect under iTerm2 / Kitty / WezTerm, and a
respectable Unicode half-block downsample under everything else
(tmux, ssh, plain xterm). It supports animated GIFs (loops in
place), reads from stdin (`curl ... | viu -`), and can pin the
output width / height to fit a layout. No GUI dependency, no
viewer launches — the image lands in the terminal scrollback
like any other text output.

## Install

```bash
# Homebrew (macOS / Linux)
brew install viu

# Cargo (any OS with a Rust toolchain)
cargo install viu

# Linux package managers
# Arch:    pacman -S viu
# Nix:     nix-env -iA nixpkgs.viu

# from a release archive (Linux x86_64)
curl -L https://github.com/atanunq/viu/releases/download/v1.6.1/viu-x86_64-unknown-linux-musl \
  -o /usr/local/bin/viu
chmod +x /usr/local/bin/viu

# verify
viu --version    # viu 1.6.1
```

## License

MIT — see
[LICENSE-MIT](https://github.com/atanunq/viu/blob/master/LICENSE-MIT).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. just show an image
viu screenshot.png

# 2. pipe from a download
curl -sL https://example.com/diagram.jpg | viu -

# 3. fit to a width (in cells), preserving aspect
viu -w 80 wide-diagram.png

# 4. fit to width AND height (will distort if both set)
viu -w 80 -h 24 banner.png

# 5. animate a GIF in place (Ctrl-C to stop)
viu cat.gif

# 6. play a GIF a fixed number of frames then exit
viu --frames 30 cat.gif

# 7. show a whole directory of images, recursively
viu -r assets/

# 8. force half-block mode (skip iTerm/Kitty detection)
viu --blocks photo.png

# 9. transparent-background PNGs render correctly under iTerm/Kitty;
#    under half-block fallback, transparent pixels become the
#    terminal background color
viu logo-transparent.png

# 10. integrate into a code-review or AI agent loop:
#     a model emits an image (matplotlib plot, mermaid render),
#     `viu plot.png` shows it inline in the same terminal session
python plot_metrics.py --out /tmp/plot.png && viu /tmp/plot.png
```

## Niche It Fills

**Image rendering in the terminal used to be three separate
problems**: a fancy renderer (iTerm's `imgcat`) tied to one
terminal, a Sixel viewer (`img2sixel`) tied to a niche protocol,
or a half-block hack that distorts colors. `viu` collapses all
three into one binary that **detects** what the host terminal
supports and picks the highest-fidelity path automatically:
Kitty graphics > iTerm inline > Sixel > half-block. The result is
a tool you can put in a script without knowing in advance which
terminal will run it.

For agent / LLM workflows where the model produces a chart or a
generated image, `viu` is the one-line "render this in the same
terminal where I'm already reading text" primitive — the agent
doesn't have to ask the user to open a browser or a file viewer.

## Vs Already Cataloged

- **Vs [`yazi`](../yazi/):** yazi is a full file manager that
  *includes* image preview as one feature; viu is a one-shot
  printer that does only image rendering. Use yazi when you want
  to browse a folder of images interactively; use viu when you
  want to print one image (or a `find -exec viu {} \;` loop) into
  the current shell.
- **Vs [`glow`](../glow/):** glow renders Markdown — and
  inline-image rendering inside Markdown is one of its features
  on supported terminals. viu is the underlying image-only
  primitive; glow uses similar protocols for the images embedded
  in a `.md` document.
- **Vs `imgcat` (iTerm2 built-in):** imgcat works *only* under
  iTerm2 (and a few derivatives). viu is the cross-terminal
  superset — same iTerm protocol output when iTerm is detected,
  Kitty graphics when Kitty is detected, half-block elsewhere —
  so a script with `viu` is portable; a script with `imgcat`
  breaks the moment someone runs it in tmux on Linux.
- **Vs Chafa:** Chafa is the long-standing C alternative with the
  widest format / protocol matrix and the best half-block /
  Sixel quality, at the cost of more flags and a bigger
  install footprint. viu is the smaller, opinionated Rust
  alternative — fewer knobs, single static binary, sane defaults.

## Caveats

- **High-fidelity output requires iTerm2, Kitty, or WezTerm.**
  Inside tmux on a plain xterm, viu falls back to half-block
  Unicode characters (`▀ ▄`) — readable for diagrams, lossy for
  photos. tmux-passthrough mode for the Kitty protocol exists
  but requires a tmux config tweak; off the shelf, tmux strips
  the graphics escapes.
- **Animated GIFs block the shell while playing.** `viu cat.gif`
  loops until you `Ctrl-C`; use `--frames N` (or `--once`) in
  scripts so the command terminates.
- **Half-block mode doubles vertical resolution but halves
  horizontal.** A 200×200 image renders into roughly 200×100
  terminal cells — wide images may overflow the screen even when
  they look fine under iTerm/Kitty. Use `-w` to constrain width.
- **No vector formats.** SVG / PDF / EPS are not supported —
  rasterize first (`rsvg-convert`, `pdftoppm`) and pipe the PNG
  in.
- **No EXIF rotation handling.** A JPEG with EXIF orientation
  metadata renders in its raw pixel order; portrait phone
  photos may show up sideways. Pre-rotate with `jpegtran` or
  `exiftran` if it matters.
