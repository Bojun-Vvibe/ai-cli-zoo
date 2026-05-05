# ascii-image-converter

> **Cross-platform CLI that converts images (and short videos /
> GIFs) into ASCII or Braille art and prints them in the terminal,
> with optional truecolor + background-fill + braille-density
> modes.** One Go binary, zero runtime deps, reads local files /
> URLs / stdin, writes to stdout / a `.txt` / a coloured `.png` /
> an `.html`. Pinned to **v1.13.1** (commit
> `d05a757c5e02ab23e97b6f6fca4e1fbeb10ab559`),
> Apache-2.0
> ([LICENSE.txt](https://github.com/TheZoraiz/ascii-image-converter/blob/master/LICENSE.txt)).

- **Repo:** https://github.com/TheZoraiz/ascii-image-converter
- **Latest version:** v1.13.1
- **License:** Apache-2.0 (`LICENSE.txt` at repo root, SPDX
  `Apache-2.0`)
- **Category:** `terminal-graphics` / `ascii-art` / `image-viewer`
- **Language:** Go

## What it does

`ascii-image-converter image.png` reads the input, downsamples it
to a character grid sized to the terminal width (`-d <W>,<H>`
overrides), maps each cell to one of the standard ASCII grayscale
ramp characters (`" .:-=+*#%@"` by default; `-m <chars>` swaps the
ramp; `--braille` switches to a Unicode braille-pattern dot
encoding that doubles horizontal and quadruples vertical resolution
per character cell), and prints the result. `-C` colours each
character with the truecolor sample of the underlying pixel so the
output looks like a low-resolution version of the source image
when the terminal supports `\033[38;2;R;G;Bm`; `-c` does the same
in 256-colour palette mode for terminals without truecolor; `-b`
fills the background with the cell colour and prints a space
(producing a chunky pixel-art look without ASCII characters at
all); `-n` (negative) and `--threshold` (braille on/off cutoff)
round out the rendering knobs.

Inputs go beyond local files: a URL is fetched directly
(`ascii-image-converter https://…/photo.jpg`), stdin is accepted
(`curl -s … | ascii-image-converter -`), and animated inputs
(`--gif`, animated PNGs, short MP4 / WebM via FFmpeg if it is on
PATH) loop the frames in the terminal at the source frame rate
until `Ctrl-C`. Outputs go beyond stdout: `--save-txt out/` writes
a plain-text version, `--save-img out/` rasterises the coloured
ASCII grid back to a PNG (high-resolution screenshot of the
terminal output without screen-capturing the terminal), `--save-html
out/` writes an HTML / CSS document for embedding in a web page or
a slide deck. Cross-platform: macOS, Linux, FreeBSD, Windows
(including the old `cmd.exe` 16-colour fallback).

## When to pick it / when not to

Pick `ascii-image-converter` when the requirement is *artistic /
text-only image rendering*: ASCII / braille art for an SSH login
banner, a `cowsay`-style splash in a script, a slide-embeddable
ASCII portrait, an `asciinema` recording where a real image would
be invisible. Pick it when the output channel is truly text — a
plain-text README, a CI log, a Slack code block — where
[`chafa`](../chafa/) / [`viu`](../viu/) sixel / kitty-image escapes
would render as noise. Pick the `--braille` mode when you want
roughly 2× horizontal × 4× vertical resolution per character cell
without leaving the text plane. Pick `--save-html` for embedding
the rendered art in static-site / deck workflows.

Skip it when the requirement is *high-fidelity in-terminal image
display* in a graphics-capable terminal — `chafa` and `viu` use
sixel / kitty-image / iTerm2-image escapes that send the actual
pixels and look photo-realistic in modern terminals (`kitty`,
`wezterm`, `foot`, `mlterm`, `iTerm2`). Skip it when you need
animated GIF playback in a graphics terminal — `chafa --animate`
and [`timg`](https://github.com/hzeller/timg) are higher fidelity
for that case. Skip it for screenshots-of-source-code (use
[`silicon`](../silicon/) / [`freeze`](../freeze/) — they
syntax-highlight code, not raster images). Skip it for thumbnail
*generation* of an image library — `imagemagick` / `vips` are the
right shape there.

Vs already cataloged: orthogonal to [`chafa`](../chafa/) /
[`viu`](../viu/) — those render images via terminal *graphics*
escapes (sixel / kitty / iTerm2) and degrade to pixel-blocks via
half-block characters when the terminal is text-only;
`ascii-image-converter` renders into the *character* plane
(ASCII / braille) and produces output that survives copy-paste,
log files, plain-text email, and CI artifact viewers — pick by
output channel. Pairs with [`figlet`](../figlet/) /
[`gowall`](../gowall/) /
[`terminaltexteffects`](../terminaltexteffects/) (typographic
banners + wallpaper recolouring + animated text effects — same
text-art neighbourhood, orthogonal verbs). Pairs with
[`hyfetch`](../hyfetch/) / [`fastfetch`](../fastfetch/) when you
want a custom ASCII logo for the system banner instead of the
shipped distro logo (convert your project mascot once, drop the
`.txt` into the fetcher's logo path).

## Example invocations

```bash
# Local file → coloured ASCII at terminal width
ascii-image-converter photo.jpg -C

# URL input, no shell-out to curl
ascii-image-converter https://example.com/logo.png -C

# Braille mode (denser dots) with truecolor + custom dimensions
ascii-image-converter portrait.png --braille -C -d 120,60

# Background-filled "pixel art" (no ASCII chars, just coloured spaces)
ascii-image-converter sprite.png -b -C -d 40,40

# Custom ramp (light → dark) for a stylised look
ascii-image-converter scene.jpg -C -m " .,:;ox%#@"

# Animated GIF — loops in terminal until Ctrl-C
ascii-image-converter funny.gif --gif -C

# Pipe in via stdin (e.g. from screenshot tools / generators)
cat artwork.png | ascii-image-converter - -C --braille

# Save the coloured ASCII grid as a PNG screenshot
ascii-image-converter mascot.png -C --save-img ./out/
# → ./out/mascot-ascii-art.png

# Save as plain-text (for committing into a repo / using in MOTD)
ascii-image-converter logo.png --save-txt ./art/

# Embed in a static site / slide deck
ascii-image-converter logo.png -C --save-html ./out/

# Verify
ascii-image-converter --version    # ascii-image-converter v1.13.1
```

## Caveats

- **Aspect ratio is character-cell-shaped.** A 1:1 image renders
  taller-than-wide because terminal cells are roughly 2:1 (height
  vs width); compensate with `-d W,H` or accept the stretch.
- **`-C` truecolor depends on the terminal.** Inside `tmux`,
  enable `set -ga terminal-overrides ",*256col*:Tc"` to forward
  truecolor; in CI logs that strip ANSI, drop `-C` and let the
  ASCII ramp carry the contrast.
- **Video / MP4 needs FFmpeg.** The image / GIF path is pure-Go
  and zero-dep; video playback shells out to `ffmpeg` for frame
  extraction — install it separately if you want that mode.
- **`--save-img` is a rasterisation of the ASCII output.** It is
  *not* a pixel-perfect copy of the input (that would just be the
  input file); it is a screenshot-like PNG of how the ASCII grid
  looks in a terminal at a chosen font.
- **No interactive viewer / pager.** The binary prints once and
  exits (or loops a GIF until `Ctrl-C`); for browsing many images
  pair with [`fzf`](../fzf/) (`fzf --preview
  'ascii-image-converter {} -C'`) or wrap in a shell loop.
