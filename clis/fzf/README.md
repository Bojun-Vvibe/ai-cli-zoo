# fzf

- **Repo:** https://github.com/junegunn/fzf
- **Version:** v0.72.0 (2026-04-26)
- **License:** MIT ([LICENSE](https://github.com/junegunn/fzf/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install fzf` · `apt install fzf` · `pacman -S fzf` · `go install github.com/junegunn/fzf@latest` · prebuilt binaries on the GitHub release page · binary name is `fzf`

## What it does

`fzf` is a general-purpose, interactive fuzzy finder that reads lines on
stdin and lets you narrow them down by typing. It is the de-facto
substrate that dozens of other terminal tools build on.

- Streams stdin into a full-screen TUI picker with sub-millisecond
  filtering on hundreds of thousands of lines, supporting exact,
  fuzzy, prefix/suffix, and inverse match operators in the query.
- Ships shell integrations for bash/zsh/fish: `Ctrl-T` (file picker),
  `Ctrl-R` (history search), `Alt-C` (cd into subdir), and a
  trigger-based completion (`**<Tab>`) that fuzz-completes paths,
  hostnames, kill targets, and git refs.
- Has a `--preview` window that runs an arbitrary shell command per
  highlighted line — e.g. `bat --color=always {}` for files,
  `git log {}` for branches — making it the glue for ad-hoc TUIs.
- Speaks `--bind` to remap any keystroke to a built-in action or shell
  command, so the same binary becomes a file browser, a git branch
  switcher, a process killer, or a docker container picker depending
  on how you wire it.
- Composes cleanly with pipes: `find ... | fzf | xargs ...` is the
  canonical "pick one and act on it" idiom for one-off CLI work.

## When to use it

Reach for `fzf` whenever you have a list and you need a human to pick
one or many items from it interactively — file paths, git branches,
processes, docker containers, kubectl resources, history entries,
TODO items, anything line-oriented. It is also the right primitive
when you're building a small bespoke TUI in a shell script and don't
want to pull in a heavier framework: `--preview`, `--bind`, and
`--header` cover most one-off workflows in five to twenty lines of
shell. The `Ctrl-R` history binding alone often justifies the
install on any developer machine.

## When NOT to use it

Skip it when the list is tiny (three or four items) — a plain `select`
prompt is shorter and has no install. Skip it when you need
**structured** picking over rich data (table view, multiple columns,
SQL-like filters) — reach for [`csvlens`](../csvlens/),
[`harlequin`](../harlequin/), or `visidata` instead. Skip it for
non-interactive pipelines where a deterministic filter is required:
`grep`, `rg`, or `awk` are reproducible; an interactive picker is
not. And skip it as the primary git-branch browser if you already
run [`gitui`](../gitui/), [`lazygit`](../lazygit/), or
[`jj`](../jj/) — those have native, faster pickers built in.
