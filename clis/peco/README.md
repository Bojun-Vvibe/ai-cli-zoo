# peco

> **Simplistic interactive filtering tool — a small Go binary that
> reads lines from stdin, lets you fuzzy-narrow them with a live
> top-bar query, and prints the selected line(s) to stdout.**
> Pinned to **v0.6.0**, MIT
> ([LICENSE](https://github.com/peco/peco/blob/master/LICENSE)).

- **Repo:** https://github.com/peco/peco
- **Latest version:** v0.6.0 (2026-02-24)
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `filter` / `tui` / `pipeline-glue`
- **Language:** Go

## What it does

`peco` is the smaller, older cousin of `fzf` / `skim`: a single
static Go binary (`brew install peco` / `go install
github.com/peco/peco/cmd/peco@latest`) that reads stdin, draws a
two-pane TUI (query line on top, scrollable matches below),
narrows the list as you type, and on Enter prints the selected
line(s) back to stdout. Composable as a unix filter:
`history | peco` jumps to a past command, `git branch | peco |
xargs git checkout` checks out a branch by fuzzy name,
`find . -type f | peco --select-1` opens the only match without
prompting. Configuration lives in `~/.config/peco/config.json`
(custom keybinds, layout `top-down` vs `bottom-up`, color
palette, multi-select with Tab) and per-context filter syntax —
including a **Migemo** mode for typing romaji to match Japanese
text — sets it apart from the more popular but
configuration-heavier alternatives. Multi-line input is
preserved, ANSI color in input is rendered, and `--exec`
replaces the selected token in a template before running it
(`ls *.log | peco --exec 'less {}'`).

## Why included

For shell glue where the *value* is "let me eyeball-narrow a
list before committing to one", `peco` is the smallest possible
dependency: one Go binary, one config file, no shell
integration required, no `~/.zshrc` snippets to source. It
predates `fzf` and remains the right pick for a script that
ships to other engineers' machines and you don't want to
require them to install + configure `fzf` first — `cat
big-list.txt | peco | xargs cmd` works the same on macOS,
Linux, and Windows the moment the binary is on PATH. For an
LLM-CLI workflow that generates a long ranked list of
candidates (test files to run, branches to inspect, log lines
to grep into) and asks the operator to pick one before the
next agent step, piping the model's output through `peco`
keeps the interactive narrowing in the terminal where the
agent can read the result back from stdout.
