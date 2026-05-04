# cbonsai

> **Generative ASCII bonsai-tree screensaver in C / ncurses** —
> a tiny C+ncursesw binary that grows a procedural bonsai in
> the terminal one branch at a time using a recursive
> branch-shoot DSL with configurable life, branch-count,
> multiplier, base-tile, leaf glyph set, and seed; runs in
> three modes (one-shot static, animated grow with
> per-step delay, or `--screensaver` infinite-loop with a
> `--wait` pause + reseed between trees), prints to a regular
> TTY (no truecolour required, 256-colour aware, falls back
> to 8-colour gracefully) — pinned to **v1.4.2** (commit
> [`06b06f0d`](https://gitlab.com/jallbrit/cbonsai/-/commit/06b06f0d23ba194a89319cd98d0ad83c5500d258),
> [LICENSE](https://gitlab.com/jallbrit/cbonsai/-/blob/v1.4.2/LICENSE),
> GPL-3.0-or-later).

Source: <https://gitlab.com/jallbrit/cbonsai>
(GitHub mirrors exist but the canonical upstream is GitLab.)

## TL;DR

`cbonsai` is the meditative-screensaver corner of the
catalog: one C binary, ~1500 lines, ncurses-only, that
draws a procedural bonsai tree in the terminal and either
stops (one-shot mode for a `figlet`-style decoration) or
loops forever with a configurable pause between trees
(screensaver mode for an idle terminal pane). The trees
are deterministic given a `--seed`, so a "show me the same
tree" is reproducible — useful for screenshots in a README
or a recorded asciinema demo.

The killer property — relative to the screensaver alternatives
in the catalog — is **deterministic seeding + a real branching
DSL**. `pipes.sh` / `cmatrix` / `unimatrix` produce moving
glyphs but no structural output; `cbonsai`'s output is a
bonsai shaped by parameters that map onto botanical
intuitions (`--life 32`, `--multiplier 5`, `--base 1` for a
small ceramic pot, `--leaf '&,*'` for a sparse foliage glyph
pair) and the same `--seed 42 --life 32` always draws the
same tree. The animated-grow mode (`--live`, with `--time
0.03` for per-step delay) is the right idle-screen
companion, and the one-shot mode (the default with
`--print`) drops a static bonsai into a shell motd.

## Install

```bash
# Homebrew (community formula, tracks upstream tags)
brew install cbonsai

# Arch Linux (extra)
sudo pacman -S cbonsai

# Debian / Ubuntu (Trixie+)
sudo apt install cbonsai

# Build from source (any Linux / BSD / macOS with ncursesw)
git clone --branch v1.4.2 https://gitlab.com/jallbrit/cbonsai
cd cbonsai
make
sudo make install         # /usr/local/bin/cbonsai

# verify
cbonsai --help | head -1  # cbonsai 1.4.2
```

Requires `ncursesw` (wide-character ncurses — present on
every modern Linux / BSD / macOS by default; the build
needs `pkg-config` to find it). One C source file, one
binary, no runtime dependencies beyond `libncursesw`.

## Example usage

```bash
# one-shot static bonsai (the default once it finishes growing)
cbonsai -p

# animated grow, slow enough to watch
cbonsai -l -t 0.03

# screensaver: grow tree, hold 4 s, reseed, repeat forever
cbonsai -S -w 4 -t 0.02

# deterministic tree (same shape every run — for README screenshots)
cbonsai -p -s 42 -L 32 -M 5

# add a typed-out poem under the pot
cbonsai -p -m "翠竹千竿映碧空"

# narrower bonsai for a sidebar tmux pane
cbonsai -p -W 30 -H 20

# print version + exit
cbonsai -V
```

Common flags:

- `-l` / `--live` animated grow (default is instant + static)
- `-S` / `--screensaver` infinite loop (Ctrl+C to exit)
- `-t SECS` / `--time` per-step animation delay
- `-w SECS` / `--wait` pause between trees in screensaver mode
- `-s N` / `--seed` deterministic RNG seed
- `-L N` / `--life` total branch lifespan (controls tree size)
- `-M N` / `--multiplier` branching density (1 sparse, 10 dense)
- `-b TYPE` / `--base` pot style (1 large ceramic, 2 small)
- `-c COLOUR` / `--leaf` leaf glyph + colour (e.g. `&,*`)
- `-m TEXT` / `--message` annotation rendered next to the pot
- `-p` / `--print` print final ASCII to stdout instead of TUI
  (useful in `motd` / `figlet`-style pipelines)
- `-V` / `--verbose` (also `--version`) version + diagnostics

## Why it matters

- **Idle terminal pane that is not noise.** The
  `pipes.sh` / `cmatrix` / `unimatrix` family is high-motion
  high-glyph noise that competes for visual attention; a
  growing bonsai in a small tmux pane reads as ambient,
  the moving glyphs are slow, and the final still frame
  is calm. The right shape for "I want a screensaver in
  pane 4 while pane 1 runs a long build."
- **Deterministic seeding** means a tree can be a stable
  decoration: pin a `cbonsai -p -s 42 -L 32` output into
  a static `motd` file, embed in a README via an
  asciinema → `agg` GIF, screenshot once and reuse forever.
- **One-shot `-p` mode** integrates into `figlet` /
  `lolcat` / `boxes`-style ASCII-art pipelines (`cbonsai
  -p | lolcat`) and into shell login scripts that want a
  small visual cue without taking over the terminal.
- **`-m TEXT` annotation** turns the bonsai into a small
  greeting card — drop a haiku, a project name, or a
  `fortune` line next to the pot for a personalised motd.
- **Renders on plain `xterm`** without truecolour, kitty
  graphics, sixel, or any terminal feature beyond
  ncursesw + 8/256 colour. Works over SSH on a
  busybox-shaped host where richer terminal-graphics tools
  do not.

## Vs Already Cataloged

- **Vs `pipes.sh` / `cmatrix` / `unimatrix` / `nyancat`:**
  same niche (idle / decorative / ambient terminal output),
  orthogonal aesthetic — those are perpetual motion of
  abstract glyphs; `cbonsai` is procedural growth ending
  in a still frame. Pick `cbonsai` for calm and structure,
  the others for "a thing is happening on this screen."
- **Vs [`tte`](../tte/):** orthogonal — `tte` is an
  *animation engine* you feed text into; `cbonsai` is a
  *generator* that produces its own content. Compose
  them with a pipe — `cbonsai -p -m "Hello"` is the input,
  `tte beams` could animate the result into place.
- **Vs [`figlet`](../figlet/) / `toilet` / `cowsay`:**
  same "ASCII art primitive" niche; orthogonal subject —
  figlet/toilet render text as banners, cowsay frames
  text in a speech bubble, cbonsai draws a bonsai. All
  compose in a `motd` script.
- **Vs `boxes`:** orthogonal — `boxes` draws frames around
  text; `cbonsai` draws a tree. The two compose:
  `cbonsai -p | boxes -d simple` for a framed bonsai.
- **Vs [`asciinema`](../asciinema/) + [`agg`](../agg/):** the
  recording surface for a `cbonsai -l` session; the
  resulting GIF is the README-embeddable artifact.

## License

GPL-3.0-or-later — see
[LICENSE](https://gitlab.com/jallbrit/cbonsai/-/blob/v1.4.2/LICENSE).
Binary use is unrestricted; redistributing modified source
carries GPL-3.0 obligations. The output (an ASCII tree) is
data, not subject to the source licence.

## Caveats

- **GPL-3.0-or-later** (not -only, not -or-2.0) — fine for
  most distributions; relevant if you embed a fork into
  a permissively-licensed product.
- **Requires `ncursesw`** (wide-character ncurses) at
  build time. Default on every modern Unix; on a
  stripped-down build environment (Alpine `-musl`
  without `ncurses-dev`) install the dev headers first.
- **No truecolour gradient support.** The leaf colours
  are 256-colour palette indices — the tree is rendered
  with a small fixed palette per leaf glyph, not
  per-pixel truecolour. Acceptable for the form factor;
  notable if you want gradient-rich aesthetics (use
  `tte` over the output instead).
- **Screensaver mode does not lock the screen.** This is
  not `xscreensaver` / `swaylock` — it is a perpetual
  drawing loop in the current terminal. For a real lock
  on a headless box use `vlock`; cbonsai is the visual
  layer, not the security layer.
- **The upstream is on GitLab, not GitHub.** Some
  package managers (older Homebrew bottles, older AUR
  helpers, Debian backports) may track stale GitHub
  mirrors — verify the version with `cbonsai -V` after
  install.

## As of

2026-05-04. Upstream tag `v1.4.2` (2025-06-19). The branching
DSL and CLI surface have been stable across the 1.x line;
the `--leaf` glyph syntax and `--message` rendering changed
between 1.0 and 1.4 — pin the version in any scripted use.
