# lazyjj

- **Repo:** https://github.com/Cretezy/lazyjj
- **Version:** v0.6.1 (commit `fe26e6764421ef6d66bd9b52c45cc72718a80e6d`,
  released 2025-09-10)
- **License:** Apache-2.0 ([LICENSE](https://github.com/Cretezy/lazyjj/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `cargo install lazyjj` · `brew install lazyjj` ·
  pre-built binaries on the GitHub releases page · Arch AUR (`lazyjj-bin`)

## What it does

`lazyjj` is a [`lazygit`](../lazygit/)-style terminal UI for the
[Jujutsu](../jj/) version-control system. Where bare `jj` is a
fast-but-flag-heavy CLI (`jj log -r 'mine() & ~empty()' -T builtin_log_compact`,
`jj squash --from @- --to @--`, `jj op log --limit 50`), `lazyjj` collapses the
common change-shaping loop into a four-pane keyboard TUI: a graph of recent
changes on the left, the diff of the highlighted change on the right, the
current bookmark / branch list and the operation log behind tab keys, and
single-letter keybindings for the workhorse verbs — `n` to start a new empty
change on top of the selected one, `e` to edit (move `@` onto) the selected
change, `s` to squash the working copy into the selected parent, `a` to
abandon, `d` to describe (open `$EDITOR` for the commit message), `P` to push
the current bookmark, `f` to fetch, `O` to open the operation log so you can
`jj op restore` your way out of a botched rebase. Every keystroke is a thin
wrapper over a real `jj` invocation, printed at the bottom of the screen — the
TUI is the *interface*, not a separate model — so what you learn in `lazyjj`
transfers directly to the CLI when you SSH into a box that has `jj` but not
`lazyjj`.

## When to pick it / when not to

Pick `lazyjj` when you've adopted `jj` (see [`jj`](../jj/)) and you spend most
of your day in the inner loop of "make a change → describe it → reorder /
squash / split → push" — a TUI that shows the change graph and the diff side
by side beats memorising `jj log -r` revsets and `jj diff -r` flags. It is
also the fastest on-ramp for a teammate who knows `lazygit` but is new to
Jujutsu's commit model: the layout maps one-to-one (graph pane =
`lazygit`'s commits, diff pane = `lazygit`'s files+diff, status bar = the
literal `jj` command being run), so the learning curve is "what do these
new verbs *mean*" not "where are they in the menu". Pairs naturally with
[`gitui`](../gitui/) (for the colocated `.git/` side of a `jj git
init --colocate` repo) and [`delta`](../delta/) (set `ui.diff.format =
"git"` + `ui.pager = "delta"` in `~/.config/jj/config.toml` for syntax-
highlighted diffs in both `jj` and `lazyjj`).

Skip it for non-Jujutsu repos — it has zero git fallback, the entire UI
assumes a `.jj/` directory exists. Skip it when scripting (`jj` itself is
the right tool for non-interactive use; `lazyjj` is interactive only). Skip
it if you actively want to *learn* the `jj` flag set — bare `jj log` /
`jj diff` / `jj rebase` will burn the syntax into your fingers faster than
a TUI that hides it. And note that the project is pre-1.0; pin the version
in your install script and review release notes before bumping minors,
because keybindings have shifted between 0.x releases.

## Example invocations

```bash
# Inside a jj-managed repo
cd my-jj-repo
lazyjj

# Common keystrokes inside the TUI:
#   ↑/↓ or j/k   move selection in the change graph
#   Tab          cycle panes (graph → diff → bookmarks → op log)
#   n            new empty change on top of the selected change
#   e            edit (move @) onto the selected change
#   d            describe (open $EDITOR for the commit message)
#   s            squash the working copy into the selected parent
#   a            abandon the selected change
#   P            git push the current bookmark
#   f            git fetch
#   O            switch to the operation-log pane (for `jj op restore`)
#   ?            show full keybinding help
#   q            quit

# Point lazyjj at a different repo without cd'ing
lazyjj --path /path/to/other/jj/repo

# Verify
lazyjj --version    # lazyjj 0.6.1
```
