# frogmouth

- **Repo:** https://github.com/Textualize/frogmouth
- **Version:** v0.9.1 (latest stable, November 2023 — last release)
- **License:** MIT ([LICENSE](https://github.com/Textualize/frogmouth/blob/main/LICENSE))
- **Language:** Python (built on Textual)
- **Install:** `pipx install frogmouth` · `pip install --user frogmouth` · `uv tool install frogmouth` · `brew install frogmouth` · binary name is `frogmouth`

## What it does

`frogmouth` is a terminal **Markdown browser** built on
[Textual](https://github.com/Textualize/textual) by the same team. It
opens a single Markdown file (`frogmouth README.md`), a directory of
them, a `file://` URL, or a remote `https://` URL (so `frogmouth
https://raw.githubusercontent.com/foo/bar/main/README.md` Just Works
without `curl` + a pipe), and renders the document as a fully
**navigable** TUI — clickable links jump in-document, between local
files, or follow remote URLs back into the same browser; an
**outline / table of contents** sidebar (`Ctrl+T`) lets you jump
between headings; a **bookmark store** (`Ctrl+B`) and per-file
**history with back / forward** (`Alt+←` / `Alt+→`) make multi-page
reading (a docs site, a multi-file repo, a folder of design notes)
behave like a real browser; and the **command palette** (`Ctrl+\`)
exposes every action as a fuzzy-searchable command. Code blocks get
syntax highlighting via `pygments`, tables render as Rich tables,
images fall back to a placeholder (or render inline on terminals
Textual supports), and the whole thing is keyboard-first with vim-ish
nav (`j/k`, `gg`, `G`, `/` for in-page search) plus mouse support.

## When to pick it / when not to

Pick `frogmouth` for **reading** Markdown — the README of a repo you
just cloned, the docs of a Python package living as `.md` in
`site-packages`, an Obsidian / Foam vault you want to browse without
launching an Electron app, a directory of meeting notes, a downloaded
copy of a spec. It is the right tool for "I'm SSH'd into a server,
let me read this README without scrolling raw text in `less`", for
browsing a repo's whole docs tree without leaving the terminal (open
the directory, click into files), and for reading remote READMEs over
the wire without cloning the repo. Pair it with [`mdcat`](../mdcat/)
when the use case is "render this Markdown into the scrollback so I
can copy-paste from it" rather than "browse it interactively" —
`mdcat` is a one-shot pretty-printer with no UI; `frogmouth` is the
browser you live in for ten minutes. Pair it with [`glow`](../glow/) if
you prefer Glow's stash / paginated style and Charm-stack ergonomics —
they overlap, and the choice is mostly aesthetic plus "do I want
bookmarks + history (frogmouth) or stash + Pretty Glamour rendering
(glow)".

Skip it for **editing** — `frogmouth` is read-only; for editing reach
for any text editor or a Markdown-aware TUI like
[`helix`](../helix/) / `nvim` with a Markdown plugin. Skip it for
**very large files** — Textual re-renders on every viewport change and
a 50 MB Markdown blob will feel sluggish; chunk it or use `less`.
Skip it for **diagrams that need real rendering** — Mermaid, PlantUML,
and embedded LaTeX render as their source code, not as an image; for
those, a real Markdown previewer in a browser-tab IDE is the right
tool. Note also that the project has been **dormant since the
v0.9.1 release in late 2023** — it still works fine on modern Python
and Textual, but if you need active maintenance or new features,
[`glow`](../glow/) and [`mdcat`](../mdcat/) are more actively shipped
and worth a look first. Finally, terminal-graphics support for inline
images depends on the host terminal supporting one of Textual's
backends — on Apple Terminal.app you'll see placeholders.

## Example invocations

```bash
# Open a single file
frogmouth README.md

# Browse a directory of Markdown (a docs/ tree, an Obsidian vault, …)
frogmouth docs/
frogmouth ~/notes/

# Open a remote README directly — no clone, no curl
frogmouth https://raw.githubusercontent.com/Textualize/textual/main/README.md
frogmouth https://raw.githubusercontent.com/sharkdp/bat/master/README.md

# Open multiple files / URLs in tabs
frogmouth README.md CHANGELOG.md docs/install.md

# Inside the TUI — keyboard cheatsheet
#   Ctrl+T          toggle table-of-contents sidebar
#   Ctrl+B          add / view bookmarks
#   Ctrl+\          command palette (every action, fuzzy-searched)
#   Alt+← / Alt+→   back / forward through visited documents
#   /               in-page search
#   j / k / gg / G  vim-ish scrolling
#   o               open a new file or URL
#   q               quit
```
