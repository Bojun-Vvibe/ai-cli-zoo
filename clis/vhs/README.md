# vhs

> **Scriptable terminal-recording engine that turns a plain-text `.tape`
> file into a deterministic GIF / MP4 / WebM of a real headless `tmux`
> session — your README demo is a diff-able artifact instead of a
> screen-capture re-shoot.** Pinned to **v0.11.0**, MIT
> ([LICENSE](https://github.com/charmbracelet/vhs/blob/main/LICENSE)).

- **Repo:** https://github.com/charmbracelet/vhs
- **Latest version:** v0.11.0
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `dev-experience` / `docs-tooling`
- **Language:** Go

## What it does

`vhs` is a headless terminal recorder driven by a tiny declarative
DSL. A `.tape` file is a sequence of directives — `Output demo.gif`,
`Set FontSize 22`, `Set Width 1200`, `Set Theme "Dracula"`,
`Type "echo hello"`, `Enter`, `Sleep 500ms`, `Ctrl+L`, `Hide` /
`Show` to bracket setup work — that `vhs demo.tape` runs against an
internal headless `tmux`, capturing every frame and encoding it via
`ffmpeg` to GIF, MP4, WebM, or animated PNG. Because the input is
text, the same tape rerun on a different machine produces a
byte-identical (or near-identical) recording: no jittery typing
speed, no accidental window resize, no "let me re-record because I
fat-fingered the prompt." A built-in `vhs serve` mode exposes a
remote SSH endpoint so CI runners without a display can render
demos, and `vhs publish demo.gif` uploads to a hosted CDN if you
don't want to commit the binary asset. Tapes can `Source` shared
prelude files (font, theme, dimensions) so a project standardises
its demo look in one place.

## Why included

Every CLI in this catalog ships a README with a hand-recorded
terminal GIF that drifts the moment the binary's output format
changes — the GIF still says `--old-flag`, the tool now wants
`--new-flag`, and nobody re-records because re-recording means
re-finding asciinema, getting the font right, and not stuttering on
the keyboard. `vhs` collapses that loop to "edit the tape, rerun in
CI, commit the new GIF beside the code change that broke the old
one," which is the only workflow that actually keeps demos honest
across a 426-entry catalog. For an LLM-CLI workflow, the same DSL
lets an agent generate a fresh demo for a tool it just learned —
write tape, render in a sandbox, attach the GIF to the PR — without
asking the human to hold a webcam to their monitor.
