# walk

> **A terminal file navigator that prints the chosen path
> on stdout** — minimal `fzf`-style file browser: launch
> `walk`, fuzzy-search and navigate the tree with arrows /
> vim keys, hit `Enter` on a file (or `Esc` on a directory),
> and the tool exits printing the selected absolute path —
> designed to be wrapped by a one-line shell function so
> `cd "$(walk)"` becomes the universal "interactively pick a
> directory and jump there" command, and `vim "$(walk)"`
> becomes "interactively pick a file and open it". Pinned
> to **v1.13.0** (SPDX: `MIT`,
> [LICENSE](https://github.com/antonmedv/walk/blob/master/LICENSE)).

Source: <https://github.com/antonmedv/walk>

## TL;DR

`walk` is a sub-megabyte single-binary Go TUI built on
`bubbletea` whose entire job is "show a navigable tree, let
the user pick one path, exit, print that path". Inside the
TUI: arrow keys / `hjkl` move; `/` enters fuzzy-search-as-you-type
and the tree filters live; `Enter` on a directory descends
into it (or on a file selects+exits); `Backspace` ascends;
`.` toggles hidden files; `:` jumps to an arbitrary path;
preview pane shows file contents (text) or directory contents
(tree). Because the chosen path is printed on stdout and
nothing else is, `walk` composes into any shell pipeline:
`cd "$(walk)"`, `vim "$(walk)"`, `tar czf out.tgz "$(walk)"`.
The README ships the canonical `lk` shell function that
makes `walk` your default `cd` replacement.

## Install

```bash
# Go install
go install github.com/antonmedv/walk@latest

# Homebrew
brew install walk

# Pre-built binary (macOS / Linux / Windows)
# https://github.com/antonmedv/walk/releases/tag/v1.13.0

# verify
walk --help
```

Add this to `~/.zshrc` / `~/.bashrc` for the universal
"navigate-and-cd" alias:

```bash
function lk {
  cd "$(walk "$@")"
}
```

## License

MIT — see
[LICENSE](https://github.com/antonmedv/walk/blob/master/LICENSE).

## Representative Commands

```bash
# 1. interactively pick a directory and cd into it
cd "$(walk)"

# 2. interactively pick a file and open it in $EDITOR
$EDITOR "$(walk)"

# 3. start at a specific subtree
walk ~/projects

# 4. fuzzy-find from the start
walk --fuzzy "config.toml"

# 5. compose into any tool that takes a path argument
tar czf snap.tgz "$(walk)"
```

## Why It Matters

Terminal file managers cluster into two ends of a spectrum:
*full applications* (`ranger`, `nnn`, `lf`, `yazi`, `xplr`,
`broot`, `joshuto`, `superfile`, `vifm`, `felix`) that you
*enter*, navigate inside, and exit; or *one-shot pickers*
(`fzf` over `find`, `fzf` over `fd`) that produce a path on
stdout for shell composition but lose the spatial / visual
tree-navigation affordance. `walk` sits exactly in the gap:
it is a real navigable tree-with-preview TUI like `ranger` /
`broot`, but it follows the unix-pipe contract — print the
chosen path on stdout, exit, do nothing else — so it is
trivially wrapped (`cd "$(walk)"`, `vim "$(walk)"`) into
whatever the surrounding shell wanted to do with the path.
The result is the most painless "interactively pick a path,
then act on it" command available: no opening a separate
file-manager process and bouncing files in and out of it
(`ranger`'s exit-cd needs a sourced wrapper, `lf` needs
`lfcd`, `yazi` needs `y`); no remembering whether `fzf`'s
preview pane is configured for tree descent (it isn't, by
default); no committing to a TUI's whole worldview. Pick
over `ranger` / `nnn` / `yazi` / `lf` when you don't want a
file-manager session — you want one path on stdout and
control to return to your shell pipeline. Pick over `fzf`
over `find` when the spatial / preview-pane / tree-descent
ergonomics matter and a flat fuzzy list does not. Pick over
`broot` when you want a tiny single-purpose binary instead
of `broot`'s broader feature surface (sizes, custom verbs,
git status). Pairs naturally with `zoxide` (use `walk` for
deep visual navigation when you don't already know the
target dir; use `z` when you do), `bat` (preview pane
already renders text), and any alias-driven shell workflow.
The killer property is **stdin-clean, stdout-pure, exits
fast** — a TUI that respects pipes.
