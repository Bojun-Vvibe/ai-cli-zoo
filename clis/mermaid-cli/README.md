# mermaid-cli

> **mermaid-cli** (`mmdc`) — the official command-line front-end to
> Mermaid: takes Mermaid diagram source (`.mmd` / `.md` fenced blocks)
> and renders SVG / PNG / PDF using a headless Chromium instance
> bundled via Puppeteer. Pinned to **v11.14.0**, MIT — license file:
> [LICENSE](https://github.com/mermaid-js/mermaid-cli/blob/master/LICENSE).

Source: <https://github.com/mermaid-js/mermaid-cli>

## TL;DR

`mmdc -i diagram.mmd -o diagram.svg` reads a Mermaid source file and
writes a rendered diagram. The same binary handles flowcharts,
sequence diagrams, class diagrams, state diagrams, ER, Gantt,
git-graph, mindmap, timeline, sankey, C4, requirement, journey,
quadrant — every diagram kind Mermaid itself supports — and accepts
themes, background colours, viewport sizes, and a custom Mermaid
config JSON to control rendering.

The killer use is **CI artefact generation**: a repo with diagrams
checked in as `.mmd` source files runs `mmdc` in CI to produce SVG
artefacts that get uploaded to docs sites or embedded in PRs, so the
*source* of the diagram lives in version control as text — diff-able,
review-able, AI-editable — and the rendered image is a build output,
not a checked-in PNG nobody can update without re-opening Lucidchart.

`mmdc` also supports input mode `-i README.md` which scans a Markdown
file for ` ```mermaid ` fenced blocks and replaces them with image
references (or rasterises each in place), making it a clean static-site
preprocessor for non-Mermaid-aware renderers.

## Install

```bash
# npm / pnpm / yarn (the canonical install path)
npm install -g @mermaid-js/mermaid-cli
pnpm add -g @mermaid-js/mermaid-cli

# Docker (avoids the Puppeteer Chromium download on the host)
docker pull minlag/mermaid-cli:11.14.0
docker run --rm -v "$PWD":/data minlag/mermaid-cli -i /data/diagram.mmd -o /data/diagram.svg

# Pre-built source distribution from GitHub
# https://github.com/mermaid-js/mermaid-cli/releases/tag/v11.14.0
```

The `npm` install pulls Puppeteer, which downloads a matched
Chromium build (~150 MB). Use `PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true`
plus `PUPPETEER_EXECUTABLE_PATH` to point at a system Chromium, or
use the Docker image to avoid the issue entirely.

## Example commands

```bash
# Render a Mermaid file to SVG
mmdc -i flow.mmd -o flow.svg

# Render to PNG with a transparent background
mmdc -i flow.mmd -o flow.png -b transparent

# Render with a dark theme at 1600×900
mmdc -i flow.mmd -o flow.png -t dark -w 1600 -H 900

# Process a Markdown file in place: replace ```mermaid blocks with images
mmdc -i README.md -o README-out.md

# Use a custom Mermaid config (theme variables, security level, etc.)
mmdc -i flow.mmd -o flow.svg -c mermaid-config.json
```

A minimal Mermaid source:

```
flowchart LR
    A[Source] --> B{Has tests?}
    B -->|yes| C[Merge]
    B -->|no| D[Reject]
```

## Niche it occupies

**Diagram-as-code renderer** for the Mermaid DSL — closest
catalogue neighbour is [`d2`](../d2/) (Terrastruct's diagram
language with its own renderer and a richer auto-layout story).

- Pick **mermaid-cli** when your team / docs / GitHub README files
  already use Mermaid (GitHub renders fenced ` ```mermaid ` blocks
  natively, so the source is *also* the canonical web preview),
  and you need the same source rendered to SVG / PNG / PDF for
  print docs, slides, or non-Mermaid-aware renderers.
- Pick [`d2`](../d2/) when the diagram is layout-heavy and Mermaid's
  Dagre / ELK auto-layout doesn't produce the shape you want — d2
  has a more powerful layout engine (TALA, ELK) and a richer style
  system, at the cost of "GitHub does not render d2 natively".
- Pick [`graphviz`](https://graphviz.org/) when the diagram is a
  pure graph and you want the historic stable layout algorithms;
  pick mermaid for sequence / Gantt / state / ER / mindmap / git-graph
  shapes graphviz wasn't designed for.
- Orthogonal to [`asciinema`](../asciinema/) / [`agg`](../agg/) /
  [`vhs`](../vhs/) (terminal-session recorders — those capture
  *runtime* artefacts; mmdc renders *static* diagrams from source).

Pairs cleanly with [`mdbook`](../mdbook/) / [`hugo`](../hugo/) /
[`zola`](../zola/) (static site generators that can call `mmdc` in
the build to materialise diagram SVGs), and with
[`marp-cli`](../marp-cli/) (slide-deck generator that itself accepts
Mermaid blocks and uses the same rendering path under the hood).

## Caveats

- Puppeteer + Chromium is the rendering engine, so the binary is
  not single-file portable — CI environments need either the
  Chromium download to succeed or the Docker image.
- Sandbox issues in containerised CI: the Docker image runs
  Chromium with `--no-sandbox` by default; mounting `/dev/shm`
  with extra space (`--shm-size=1g`) avoids tab crashes on large
  diagrams.
- Versions are pinned to a specific `mermaid` core version; a
  diagram syntax that works in mermaid `11.x` may render
  differently or fail in `10.x`. Pin both `mmdc` and the
  `mermaid` upstream version your docs target.
- No incremental render mode — every invocation re-launches
  Chromium. For watch-mode rendering during authoring, the
  VS Code Mermaid extension or the live editor at
  mermaid.live is the better surface.

## Citation

- Repo: <https://github.com/mermaid-js/mermaid-cli>
- Latest release: **v11.14.0**
- License: **MIT**
- License file: [LICENSE](https://github.com/mermaid-js/mermaid-cli/blob/master/LICENSE)
