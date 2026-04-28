# presenterm

- **Repo:** https://github.com/mfontanini/presenterm
- **Version:** v0.16.1 (latest stable, 2026-02)
- **License:** BSD-2-Clause ([LICENSE](https://github.com/mfontanini/presenterm/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install presenterm` · `cargo install presenterm` · `nix run nixpkgs#presenterm` · binary releases on the GitHub release page · `winget install presenterm` (Windows)

## What it does

`presenterm` is a terminal slideshow tool that renders a single Markdown file
as a full-screen presentation right inside your terminal — no browser, no
Electron, no PowerPoint. Slides are separated by `---` (the standard Markdown
horizontal rule) and the renderer handles the surprising-for-a-TUI long tail:
fenced code blocks with syntax highlighting (~250 languages via `syntect`),
images rendered through the kitty / iterm2 / sixel graphics protocols (so PNG
and JPEG actually display, not as ASCII art), inline LaTeX math via `typst`
or `mathjax`, Mermaid diagrams (`mermaid-cli` shells out at render time),
two-column layouts, speaker notes (broadcast over a local socket to a second
`presenterm --speaker-notes` window so the audience screen stays clean),
incremental reveals (`<!-- pause -->` between bullets), code highlight
animation (`{1-5|6,8|9-12}` Asciinema-style line ranges), and embedded
[`vhs`](../vhs/) tape execution for live-looking demos that are actually
deterministic. A YAML front-matter block (`title:`, `theme:`, `author:`,
`options.implicit_slide_ends:`) configures the deck; themes are also YAML and
can be vendored per-repo. The hot-reload mode (`presenterm -X file.md` or
hitting `r`) re-renders on save, so you write slides in your normal editor
and the terminal updates as you type. Export to PDF goes through `weasyprint`
or `chromium`; export to HTML is built in.

## When to pick it / when not to

Pick `presenterm` when you are a CLI-native engineer giving a technical talk
and the audience needs to see real code, real shell output, and real diffs —
the integration with terminal graphics protocols means a screenshot of your
production Grafana dashboard or an architecture PNG renders inline at full
fidelity, not as Unicode block art. The speaker-notes-over-socket trick
removes the eternal "presenter view vs main view" hardware dance: open two
terminal windows on the same machine (or one on each monitor), point one at
`--speaker-notes-mode publisher` and one at `listener`, done. Pair with
[`vhs`](../vhs/) when the demo command takes 30s of network round trips you
do not want to live-perform — the tape gives you a reproducible recording
that still *looks* live. The Markdown source diffs cleanly in code review,
which is the differentiator versus any binary slide format: a 300-line deck
PR is reviewable line-by-line, themes can be enforced via lint, and slides
live next to the code they describe.

Skip it for non-technical audiences who expect Keynote / PowerPoint
production values (animated transitions between every slide, video embeds
with audio, bezier-curve shapes, click-to-build infographics) — `presenterm`
is intentionally typographic, not motion-graphic. Skip it on terminals
without a graphics protocol (most stock VTE / xterm builds without sixel,
the macOS default Terminal.app, ssh through a chain that strips escape
sequences) — images degrade to "image not supported" placeholders, which is
worse than a real slide tool. Skip it for collaborative editing workflows
that need real-time multiplayer cursors (Google Slides territory). For
docs-as-presentation hybrids where the same Markdown becomes both a deck
*and* a published article, [`marp`](https://marp.app/) (not in this catalog)
trades terminal rendering for HTML/PDF export polish; for live in-terminal
demos with rewindable command output, [`asciinema`](https://asciinema.org/)
records, and [`vhs`](../vhs/) scripts — `presenterm` slots between them as
the slide layer.

## Example invocations

```bash
# Run a deck in the current terminal (vim keys to navigate, q to quit)
presenterm slides.md

# Hot-reload on file save while you write
presenterm -X slides.md

# Dump every slide as a PNG (uses the configured terminal graphics protocol)
presenterm --export-pdf slides.md         # -> slides.pdf
presenterm --export-html slides.md        # -> slides.html

# Validate themes and front matter without rendering
presenterm --validate-overrides slides.md

# Two-window speaker-notes mode (run each in its own terminal)
presenterm --speaker-notes-mode publisher slides.md   # audience screen
presenterm --speaker-notes-mode listener  slides.md   # presenter screen

# List the bundled themes
presenterm --list-themes
```

Minimal `slides.md` for reference:

```markdown
---
title: Why we ripped out the legacy queue
author: Bojun
theme:
  name: dark
options:
  implicit_slide_ends: true
---

# Why we ripped out the legacy queue

<!-- speaker_note: open with the on-call horror story, 60s max -->

---

## The 3am page that started it

```bash
$ kubectl logs queue-worker-7f9 | tail -20
panic: runtime error: index out of range [42] with length 0
```

<!-- pause -->

- 17 outages in Q3
- p95 lag: 4 minutes (SLO: 5 seconds)
- two engineers full-time on babysitting

---

## What we shipped instead

![architecture](./diagrams/new-pipeline.png)
```
