# terminaltexteffects

> **A terminal visual-effects engine that animates arbitrary
> text — your `cat README.md`, your `figlet` banner, your build
> success message — through one of ~30 named effects (matrix
> rain, decrypt, beams, slide, burn, rain, spotlights, wipe,
> binarypath, swarm, …)** as a CLI, a library, or a stage in a
> shell pipeline. Pure Python, no `curses` dependency on
> Windows, ANSI 8/256/truecolor adaptive. Pinned to
> **release-0.14.2** (SPDX: `MIT`,
> [LICENSE](https://github.com/ChrisBuilds/terminaltexteffects/blob/main/LICENSE)).

Source: <https://github.com/ChrisBuilds/terminaltexteffects>

## TL;DR

`tte` is a stdin-aware filter:
`echo "Deploy succeeded" | tte slide` decodes the input as a
2D grid of characters and animates it onto the terminal using
the chosen effect's per-character motion / color / timing
script. Each effect is a first-class subcommand with its own
flags (`--final-gradient-stops`, `--movement-speed`,
`--noise-colors`, `--ramp-symbols`, …) so the same library that
powers a 1-second "build green" badge can drive a 30-second
intro splash with the same code path.

Library mode: `from terminaltexteffects.effects import
effect_matrix; effect_matrix.Matrix("hello").terminal_output()`
— every effect is a class you instantiate with the input
string, so a CI dashboard, a TUI welcome screen, or a
[`textual`](https://github.com/Textualize/textual) widget can
embed an effect without spawning a subprocess.

Effect catalogue includes: `beams`, `binarypath`, `blackhole`,
`bouncyballs`, `bubbles`, `burn`, `colorshift`, `crumble`,
`decrypt`, `errorcorrect`, `expand`, `fireworks`, `highlight`,
`laseretch`, `matrix`, `middleout`, `orbittingvolley`,
`overflow`, `pour`, `print`, `rain`, `random`, `rings`,
`scattered`, `slice`, `slide`, `spotlights`, `spray`, `swarm`,
`sweep`, `synthgrid`, `unstable`, `vhstape`, `waves`, `wipe`.

## Install

```bash
# pipx (recommended — isolated env, CLI on PATH)
pipx install terminaltexteffects

# pip
pip install --user terminaltexteffects

# Browse all effects with previews
tte --help
tte <effect-name> --help
```

## Usage

```bash
# Animate stdin
echo "Deploy succeeded" | tte slide

# Animate a file
cat banner.txt | tte matrix

# Pipe through any text producer
figlet "v1.2.3" | tte rain --rain-symbols "⋅•∘○●"

# Composable in CI / scripts (one-shot, then exits)
make test && echo "ALL GREEN" | tte burn

# Library use
python -c "
from terminaltexteffects.effects import effect_decrypt
e = effect_decrypt.Decrypt('access granted')
for frame in e:
    print(frame)
"
```

## Why it matters

Terminal output is the most-watched surface in a developer's
day, and 99% of it is plain monospace ASCII. The cost of
making a *meaningful* moment (deploy succeeded, build broken,
release shipped, intro for a recorded demo, talk-opener
slide) visually distinct from the noise has historically been
"open Keynote" or "render a GIF in After Effects" — friction
that kills the use case. `tte` collapses that to a one-line
shell pipeline and keeps the output inside the terminal where
it composes with [`asciinema`](../asciinema/) /
[`vhs`](../vhs/) / [`agg`](../agg/) recordings, with `tmux`
panes, with SSH sessions, and with CI logs (one-shot mode
exits cleanly so a GitHub Actions step renders a static final
frame in the captured log). Pairs with [`figlet`](../figlet/)
/ [`gum`](../gum/) / `cowsay` / [`ponysay`](../ponysay/) /
[`boxes`](../boxes/) (text producers — `tte` is the animator
that consumes their output), with
[`chafa`](../chafa/) / [`viu`](../viu/) (image-shaped
counterparts — `tte` is the *text*-shaped animator), and is
orthogonal to TUI frameworks like
[`textual`](https://github.com/Textualize/textual) /
[`bubbletea`](https://github.com/charmbracelet/bubbletea) which
build *interactive applications* — `tte` does *non-interactive
typographic animation* and is composable inside those
frameworks as a startup splash. Caveats: requires a truecolor
terminal for the gradient-rich effects (most modern terminals
qualify), and the animations are deliberately *blocking* on
the foreground — wrap in `&` or run in a dedicated pane if you
want them not to delay subsequent commands.
