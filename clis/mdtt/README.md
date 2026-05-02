# mdtt

> **An interactive TUI editor for Markdown tables** — paste a
> table, edit cells with vim-like motions, sort columns,
> add/delete rows, and emit perfectly aligned GFM Markdown back
> to stdout. Pinned to **v0.1.4**
> ([LICENSE](https://github.com/grcsahin/mdtt/blob/master/LICENSE),
> MIT).

Source: <https://github.com/grcsahin/mdtt>

## TL;DR

`mdtt` (Markdown Table Tool) solves the exact problem every
Markdown writer has had: editing a table by hand in a plain-text
editor is miserable — you re-pad pipes after every cell change,
the alignment colons drift, and adding a column is a 14-line
search-and-replace. `mdtt` is a Bubble Tea TUI that takes a
Markdown table on stdin (or `-f file.md`), renders it as a
spreadsheet-style grid, and lets you navigate with `hjkl`,
edit cells in-place, insert / delete rows + columns with single
keystrokes, sort by column, and emit a re-aligned GFM table back
to stdout when you press `q`. It is tiny (single Go binary,
~5 MB), opinionated (GFM only — no MultiMarkdown / reST), and
built for the pipe: `pbpaste | mdtt | pbcopy` is the canonical
workflow on macOS.

## Repo + version + license

- Repo: <https://github.com/grcsahin/mdtt>
- Latest release: **v0.1.4**
- License: **MIT** —
  <https://github.com/grcsahin/mdtt/blob/master/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `master`
- Language: Go (Bubble Tea / Lip Gloss)

## Install

```bash
# go install
go install github.com/grcsahin/mdtt@latest

# Homebrew tap (community)
brew install grcsahin/tap/mdtt

# from source
git clone https://github.com/grcsahin/mdtt && cd mdtt
go build -o mdtt . && sudo install -m 0755 ./mdtt /usr/local/bin/

# verify
mdtt --version    # v0.1.4
```

## Examples

```bash
# edit a table from a file, write back in place
mdtt -f docs/api.md

# pipeline mode — paste from clipboard, edit, copy back (macOS)
pbpaste | mdtt | pbcopy

# pipeline mode — Linux (xclip)
xclip -selection clipboard -o | mdtt | xclip -selection clipboard

# inside the TUI:
#   h j k l     — move cell cursor
#   i / Enter   — edit cell (text input mode)
#   Esc         — exit edit mode
#   o / O       — insert row below / above
#   a / A       — insert column right / left
#   dd / dc     — delete row / delete column
#   s           — sort by current column (toggle asc/desc)
#   :           — set alignment (left | center | right)
#   q           — render to stdout and exit
#   Ctrl-C      — abort without writing
```

Sample input → output:

```markdown
| name | runtime | stars |
|---|---|---|
| ripgrep | rust | 50000 |
| fd | rust | 35000 |
| bat | rust | 50000 |
```

After `s` on the `stars` column (desc) and `a` to add a `lang` column:

```markdown
| name    | runtime | stars | lang |
| ------- | ------- | ----- | ---- |
| ripgrep | rust    | 50000 | rs   |
| bat     | rust    | 50000 | rs   |
| fd      | rust    | 35000 | rs   |
```

## Why it matters

Editing Markdown tables is the one place where every Markdown
editor (VS Code, Obsidian, Typora, Zed) ships a half-broken
"reformat table" command. `mdtt` is the standalone, terminal-
native, pipe-friendly tool that does *only* this one thing
correctly — including alignment colons (`:---`, `:---:`, `---:`),
embedded inline code (backticks inside cells), and Unicode
width (CJK / emoji cells get the right padding). It is the
moral equivalent of `jq` for Markdown tables: a small composable
filter you wire into your editor's "pipe selection through
shell command" feature.

## Comparison

| Tool                    | Interactive | Pipe-friendly | Sort     | Add/del col    | Unicode-width safe |
| ----------------------- | ----------- | ------------- | -------- | -------------- | ------------------ |
| `mdtt`                  | yes (TUI)   | yes           | yes      | yes            | yes                |
| VS Code Markdown ext    | yes (GUI)   | no            | no       | manual         | partial            |
| `prettier --parser md`  | no          | yes           | no       | no (reformat)  | yes                |
| `pandoc` table writer   | no          | yes (convert) | no       | no             | yes                |
| Obsidian Advanced Tables| yes (GUI)   | no            | yes      | yes            | yes                |

## License

- License: **MIT**
- Path in repo: `LICENSE`
- URL: <https://github.com/grcsahin/mdtt/blob/master/LICENSE>
