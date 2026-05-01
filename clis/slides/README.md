# slides

## What it does
A terminal-based presentation tool that renders a single Markdown file as a slide deck inside the terminal, using `---` as the slide separator. It uses `glamour` for inline Markdown rendering (so headings, lists, code fences, tables, links, and emphasis all keep their styling) and supports a `<!-- ... -->` HTML-comment block on each slide for speaker notes that are streamed to a separate pipe. A slide can also embed shell commands inside fenced code blocks tagged with a magic language hint (e.g. ` ```bash\n... ``` ` followed by `<!-- pre -->` / `<!-- exec -->`) and `slides` will *execute* the block when you press `e` and inline the output back into the slide — which turns a deck into a live demo without leaving the TUI.

## Why it's interesting
Different shape from `presenterm` (richer feature set, image rendering, Mermaid; heavier binary, more config) and from `marp-cli` / `reveal.js` / `Beamer` (browser- or PDF-targeted, not terminal-native). `slides` is a single ~6 MB Go binary, takes one positional argument (the `.md` file), has no config file, and renders deterministically on any terminal that supports 256 colors — so it ships in a `Dockerfile` for a kiosk / pairing session in one line, and a presentation lives as one Markdown file in the repo it documents.

## Niche category
Terminal presentation — single-file Markdown deck renderer with optional live shell-exec slides.

## Repo
https://github.com/maaslalani/slides

## Version pinned
`v0.9.0`

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
brew install slides
# or
go install github.com/maaslalani/slides@v0.9.0
# or grab a prebuilt binary from the v0.9.0 release page
```

## Usage examples
```sh
# Run a deck:
slides deck.md

# Minimal deck.md (3 slides, --- separators):
#   # Title slide
#   ---
#   ## Bullet slide
#   - point one
#   - point two
#   ---
#   ## Live demo
#   ```bash
#   uname -a
#   ```
#   <!-- exec -->
#
# Inside the TUI:
#   j / k or arrow keys / space = next / prev slide
#   e                           = execute the current slide's code block
#   q                           = quit
```
