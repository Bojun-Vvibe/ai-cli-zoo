# tte (terminaltexteffects)

> **Terminal visual effects engine** — a Python library + `tte`
> CLI that renders text with ~25 named visual effects (rain,
> beams, decrypt, scattered, slide, spray, swarm, wipe, …) on
> any ANSI-capable terminal: feed it text on stdin or as an
> argument, pick an effect, and characters animate into their
> final position frame-by-frame using truecolour gradients,
> easing curves, and per-character motion paths — pinned to
> **release-0.14.2** (commit tag `release-0.14.2`,
> [LICENSE](https://github.com/ChrisBuilds/terminaltexteffects/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ChrisBuilds/terminaltexteffects>

## TL;DR

Most "terminal animation" tools are either toy screensavers
(`pipes.sh`, `cmatrix`) or single-effect demos. `tte` is the
*engine*: a documented Python API (`from terminaltexteffects.
effects import effect_rain`) plus a thin CLI (`tte rain`,
`tte decrypt`, `tte beams`, …) that lets you compose the same
effect into a script, a tutorial intro, a build-step banner, an
asciinema cast for a README, or a docs site terminal demo.
Every effect takes a uniform set of flags — gradient stops,
animation rate, easing function (`IN_QUAD`, `OUT_BACK`,
`IN_OUT_SINE`, …), final-state colour — so the same `cat
banner.txt | tte beams --beam-gradient-stops ...` invocation
gives a deterministic output you can render in CI to a GIF.

The killer property is the **library + CLI parity**: anything
the CLI does, your Python script can do programmatically with
the same parameters, so a tutorial author prototypes the look
in the shell, then drops the equivalent `EffectIterator` into
their docs build script with no re-tuning.

## Install

```bash
# pipx (recommended — keeps the deps off your global env)
pipx install terminaltexteffects

# pip
pip install terminaltexteffects

# uv
uv tool install terminaltexteffects
```

Pure Python, no compiled deps. Requires Python 3.9+. The
binary is named `tte`.

## Example usage

```bash
# rain effect on a banner
figlet "ship it" | tte rain

# decrypt-style reveal
echo "the secret is: there is no secret" | tte decrypt

# beams effect with a custom truecolour gradient
cat README.md | tte beams \
  --beam-gradient-stops 8A2BE2 4B0082 00FF7F \
  --beam-gradient-steps 12 \
  --final-gradient-stops FFFFFF

# list all effects
tte --help

# per-effect knobs
tte rain --help
```

Capture as a recording for a README:

```bash
asciinema rec demo.cast -c 'figlet "ship it" | tte rain'
agg demo.cast demo.gif
```

## Why it matters

Terminal-first projects need a way to ship a *visual* artifact
(README hero GIF, docs site demo, conference talk intro) without
leaving the terminal — the alternative is a hand-edited Keynote
slide that drifts from reality. `tte` makes the "render the
real output of my CLI with a one-line decorative effect" path
a single shell pipe, and the deterministic flags mean the GIF
is reproducible from `make docs`. Pairs with
[`vhs`](../vhs/) (record terminal sessions to GIF/WebM/MP4 from
a `.tape` script — `vhs` for the recording surface, `tte` for
the in-terminal animation that gets recorded), with
[`agg`](../agg/) (asciinema cast → GIF — same role as `vhs` for
casts you already have), with [`gum`](../gum/) (interactive
shell-script TUI primitives — `gum style` for static styled
text, `tte` for animated text), and with
[`figlet`](../figlet/) / [`freeze`](../freeze/) on the static-
text composition side (build the text with `figlet`, animate it
with `tte`, recordthe terminal with `vhs`).

## License

MIT. See
[LICENSE](https://github.com/ChrisBuilds/terminaltexteffects/blob/main/LICENSE)
in upstream.

## As of

2026-05-04. Upstream tag `release-0.14.2` (latest GitHub release
on `ChrisBuilds/terminaltexteffects` as of this snapshot).
Effect names and flag schemas may change across minor versions;
re-check before pinning in CI.
