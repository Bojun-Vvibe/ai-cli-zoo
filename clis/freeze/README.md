# freeze

> Snapshot date: 2026-04. Upstream: <https://github.com/charmbracelet/freeze>

charmbracelet's single-binary Go tool for **generating beautiful
screenshots of source code and terminal output** as PNG / SVG / WebP
without opening a browser, screenshot app, or website like Carbon /
Ray.so. Point it at a file (`freeze main.go`) or pipe a command into
it (`freeze -x "ls -la"`), get a publication-ready image with
syntax highlighting, window chrome, padding, shadows, and rounded
corners — all configurable, all reproducible from a config file
checked into the repo.

It is the CLI answer to "I need a clean code screenshot for a blog
post / a slide / a PR description" that has historically required a
hosted web service and a manual paste.

## 1. Install

- `brew install charmbracelet/tap/freeze`
- `go install github.com/charmbracelet/freeze@latest`
- `nix-env -iA nixpkgs.freeze`
- `pacman -S freeze`, `yay -S freeze-bin`
- Standalone tar.gz / .deb / .rpm / .apk per platform on the
  [releases page](https://github.com/charmbracelet/freeze/releases)
- Single static Go binary, ~6 MB, no runtime dependencies. Headless
  rendering via `resvg` — no Chromium / Node / browser involved.

## 2. Version pin

**v0.2.2** (released 2025-04-01). Verify with `freeze --version`.

## 3. License

MIT. SPDX: `MIT`. Full text at
[`LICENSE`](https://github.com/charmbracelet/freeze/blob/main/LICENSE)
in the repository root.

## 4. What it does

Two input modes:

- **File mode** — `freeze main.go -o out.png` reads a source file,
  syntax-highlights it via Chroma against any of ~70 languages and
  ~50 themes, and renders the result as PNG / SVG / WebP.
- **Execute mode** — `freeze -x "git log --oneline -5"` runs a
  command, captures its ANSI output (including 24-bit colour, bold,
  italic), and renders the terminal frame as an image.

Output is fully styled:

- Themes (`--theme dracula | nord | catppuccin-mocha | github-dark
  | monokai | ...`) match the popular editor / terminal palettes.
- Window chrome: `--window` toggles the macOS-style traffic-light
  bar; `--border.radius`, `--border.width`, `--border.color` shape
  the frame; `--shadow.blur`, `--shadow.x`, `--shadow.y` add a
  drop-shadow.
- Margins, padding, font, font-size, line-height, line numbers
  (`--show-line-numbers`), and per-line highlighting (`--lines
  3,5-9`) are all flags.
- Configuration can live in a `freeze.json` next to the file so
  every screenshot in a docs directory looks identical without
  re-typing 12 flags.

## 5. Install & first use

```bash
brew install charmbracelet/tap/freeze
freeze --help

# code → png with line numbers
freeze main.go --show-line-numbers --theme catppuccin-mocha -o code.png

# command output → svg (vector, scales cleanly in slide decks)
freeze -x "tree -L 2" --output tree.svg

# generate a base config and reuse it
freeze --config base > freeze.json
freeze --config freeze.json README.md
```

## 6. Example: reproducible docs screenshots

```bash
# in a Makefile target for a docs directory
freeze --config docs/freeze.json src/parser.go \
  --lines 12-34 \
  --output docs/img/parser.png

freeze --config docs/freeze.json \
  -x "go test ./... -run TestParser -v" \
  --output docs/img/parser-test.png
```

A `freeze.json` checked into the repo means every contributor
regenerates identical screenshots on every release; no more drift
between blog post and current code.

## 7. Why it matters in this catalog

Several CLIs in this catalog (`vhs`, `asciinema`, `presenterm`,
`silicon`) sit in the "produce visual artefacts of terminal work"
niche; freeze is the static-image, headless, scriptable member of
that family. For an AI agent that needs to attach a snippet of code
or a command's output to a PR description / a Slack thread / a
status doc, `freeze -x "..." -o out.png && gh pr comment 1234 --body
"$(cat msg.md)"` is a reproducible primitive that does not require
a screenshot tool, a hosted web app, or a browser.

It is also the right fallback when [`vhs`](../vhs/) (motion GIFs of
terminal sessions) is overkill — a single still frame is faster to
produce, smaller to host, and renders cleanly in markdown.

## 8. Alternatives

- [`silicon`](../silicon/) — the original Rust "code → PNG"
  generator; smaller theme set, no execute mode, no SVG output.
- [`vhs`](../vhs/) — sister Charm tool for **animated** terminal
  recordings; pair the two on the same `.tape` / `freeze.json`
  layout for parallel still + GIF renders of the same demo.
- Carbon, Ray.so, polacode — the hosted-web / VS-Code-extension
  equivalents that freeze replaces for terminal-only workflows.

## 9. Telemetry

No analytics SDK in the binary. The only network access is the
optional fetch of a Google-Fonts WOFF on first run when you ask for
a font that is not installed locally; pass `--font.family <local>`
to keep it fully offline.
