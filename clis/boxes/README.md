# boxes

> **Filter that draws ASCII / Unicode art boxes around your
> text** — pipe-friendly, ships ~60 box designs (C-comment
> banners, shell `#`-banners, Unicode rounded, dog cartoon,
> peek-a-boo cat, "stone" 3D, ANSI color frames), and
> *removes* boxes too so it round-trips inside an editor.
> Pinned to **v2.3.1**
> ([LICENSE](https://github.com/ascii-boxes/boxes/blob/master/LICENSE),
> GPL-3.0-or-later).

Source: <https://github.com/ascii-boxes/boxes>

## TL;DR

`boxes` is a 25-year-old C utility (originally Thomas Jensen,
1999, still actively maintained — 2.3.1 was cut October 2025)
that does exactly one thing well: it reads text on stdin,
wraps it in a configurable border, and writes the result to
stdout. The killer feature is the **box design file** (`boxes-
config`): a declarative format describing each border as a
grid of "shapes" (NW corner, N edge, NE corner, etc.) so a
new design is ~30 lines of config, no recompile, and the
shipped library covers C/C++ block comments, shell/Python `#`
banners, HTML `<!-- -->`, SQL `--`, Lisp `;;`, ASCII boxes
(`+--+`), Unicode boxes (single / double / heavy / rounded),
plus joke designs (a cat, a dog, an "ice cube", a stone
tablet). Crucially, `boxes -r` *removes* a box — so binding
`,mc` in vim to "send paragraph through `boxes -d c-cmt2` to
add a banner" and `,md` to "send through `boxes -r` to
strip it" gives you a fully reversible commenting workflow
inside any editor that pipes through filters. ANSI color
support (since 2.x) means the box itself can be colored
without the colors leaking into the wrapped content. Width,
padding, alignment (left / center / right within the box),
and indentation are all flag-controlled.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install boxes

# Debian / Ubuntu
sudo apt install boxes

# Fedora / RHEL
sudo dnf install boxes

# Arch
sudo pacman -S boxes

# Single-binary download (GitHub releases, v2.3.1)
curl -LO https://github.com/ascii-boxes/boxes/releases/download/v2.3.1/boxes-2.3.1-musl-x86_64
chmod +x boxes-2.3.1-musl-x86_64
sudo mv boxes-2.3.1-musl-x86_64 /usr/local/bin/boxes

# Build from source pinned to release tag
git clone --depth 1 --branch v2.3.1 https://github.com/ascii-boxes/boxes.git
cd boxes && make && sudo make install
```

## Usage

```bash
# Default design (varies by build; usually 'stone')
echo "Hello there" | boxes

# Pick a design + center the text inside a 60-wide box
echo "Release v3.4.0" | boxes -d shell -a c -s 60

# Wrap a multi-line release-note paragraph into a C block-comment banner
boxes -d c-cmt2 < notes.md > notes.c

# Strip a box back out (inverse of -d)
boxes -r < notes.c > notes.md

# List every available design with a tiny preview
boxes -l

# Use ANSI color (requires a design that declares color shapes)
echo "DANGER" | boxes -d nuke -p a4

# Inside vim: visually select lines, then
# :'<,'>!boxes -d shell -a c
# wraps the selection in a centered shell banner; -r removes it.
```

Common flags worth knowing: `-d <design>` picks the design,
`-s WxH` forces minimum size, `-p a<N>` adds N spaces of
padding all around (or `l<N>r<N>t<N>b<N>` per-side), `-a
hcvc` aligns horizontally and vertically, `-i text|box|none`
controls indentation handling, `-r` removes, `-l` lists
designs, `-c <shape>` overrides a single border character on
the fly without writing a config.

## Why it's interesting

The "decorate text in a terminal" niche has fragmented into
loosely-related tools that each solve a slice:
[`figlet`](https://github.com/cmatsuoka/figlet) /
[`toilet`](https://github.com/cacalabs/libcaca) (large ASCII
*letters*, no wrapping), [`cowsay`](https://github.com/cowsay-org/cowsay)
(speech bubbles around fixed cartoon templates, no design
DSL), [`gum style`](../gum/) (Bubble Tea–era styled boxes
with rounded Unicode borders, but no inverse / remove
operation and no comment-marker designs), and `boxes`
itself. `boxes` is the only one in the set that treats the
border as a **reversible markup transform on a paragraph of
real content** — that property is why it's been the
canonical "wrap this comment in a banner" tool in vim /
emacs / Sublime keybindings since the late 90s. The design-
DSL means a corporate team can ship a one-file site-wide
config with the company banner style, and it's instantly
available everywhere `boxes` is installed; nothing in the
modern `gum`/`charm` stack offers that. Pick `boxes` when
(a) you want banner comments inside source files that you
will later need to edit or strip without re-typing them, (b)
you want a single-binary, dep-free filter that works in any
pipeline (CI logs, build banners, README ASCII art) with no
runtime, or (c) you maintain dotfiles / scripts where
"announce the next phase" should be a one-liner like
`echo "Building" | boxes -d shell`. Skip it for modern TUI-
chrome use cases (rounded styled boxes inside a Bubble Tea
app — that's [`gum style`](../gum/)'s job) or for *letter*
art where you want the text itself to be 8 lines tall (use
`figlet` / `toilet`, often piped *into* `boxes` for a
combined effect: `figlet hello | boxes -d stone`). Stable,
packaged in every major distro, MIT-of-the-old-school
mindset: the tool, the format spec, and the shipped design
library have been backwards-compatible for two decades.
