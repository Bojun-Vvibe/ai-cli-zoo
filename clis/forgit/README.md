# forgit

> **`fzf` × `git` for every command that benefits from
> fuzzy-picking** — a shell plugin that wraps the
> high-friction git verbs (`add`, `reset`, `checkout`,
> `stash`, `log`, `diff`, `cherry-pick`, `rebase -i`,
> `branch -d`, `clean`) with `fzf` multi-select pickers and
> live `delta` / `bat` previews, so "stage hunks
> interactively", "checkout a branch from a list of 200",
> "drop one stash by name", or "interactively rebase the
> last 30 commits" stop being multi-step rituals and become
> one short alias (`ga`, `gco`, `gcb`, `gcp`, `grb`, …)
> that opens a picker. Pinned to **26.05.0** (SPDX: `MIT`,
> [LICENSE](https://github.com/wfxr/forgit/blob/master/LICENSE)).

Source: <https://github.com/wfxr/forgit>

## TL;DR

`forgit` is a sourced shell plugin (bash / zsh / fish) that
exposes a family of `git_*` functions and short aliases
backed by `fzf`. Type `ga` instead of `git add -p` and a
multi-select picker of unstaged paths opens with a hunk-level
diff preview (rendered through `delta` if installed); `glo`
opens the commit log with a side preview of the patch and
lets you fuzzy-pick a SHA to copy / cherry-pick; `gss` shows
your stashes with their diff in the preview pane and `Enter`
applies one; `gcb` lists local *and* remote branches with
the latest commit message in preview and checks out the
chosen one; `grh` reset-picks files from `HEAD`; `grb`
opens an interactive rebase picker. Every command honours
`FORGIT_*` environment variables for paging, fzf options,
and which pager (`delta`, `diff-so-fancy`, `bat`) renders
the preview.

## Install

```bash
# Homebrew (installs as a zsh / bash plugin path)
brew install forgit

# zinit / zsh
zinit load wfxr/forgit

# Oh-My-Zsh custom plugin
git clone https://github.com/wfxr/forgit.git \
  ~/.oh-my-zsh/custom/plugins/forgit
# then add `forgit` to plugins=(...) in ~/.zshrc

# verify after sourcing
type ga   # ga is a shell function
```

## License

MIT — see
[LICENSE](https://github.com/wfxr/forgit/blob/master/LICENSE).

## Representative Commands

```bash
# 1. interactively stage hunks (multi-select with TAB, preview shows the hunk)
ga

# 2. fuzzy checkout from local + remote branches with last-commit preview
gcb

# 3. fuzzy cherry-pick from any branch's log into the current branch
gcp main

# 4. interactively rebase the last 30 commits, picking edit/squash/drop in the picker
grb

# 5. browse the commit log, preview the patch, copy a SHA on Enter
glo
```

## Why It Matters

Most of `git`'s daily-use friction is not the operation
itself — it is the *selection* step before the operation:
"which 3 of these 17 modified files do I want to stage",
"which of these 84 branches did I create yesterday", "which
of these 12 stashes was the WIP from before lunch", "which
30-commit window do I want to interactively rebase". `git`
ships these as text-list-plus-flag UIs (`git add -i`'s
numbered prompts, `git branch | grep`, `git stash list` then
counting), and every team eventually papers over them with
half-baked aliases that re-implement a worse `fzf`. `forgit`
is the polished version of that paper-over: every selection
step becomes an `fzf` multi-select picker with a real diff
preview, so the cognitive distance from "I want to do X to
some subset of git's state" to actually doing it collapses
to one alias. Pick over raw `git add -p` / `git checkout
<TAB>` when interactive selection is the actual time sink;
pick over `lazygit` / `gitui` when you want fuzzy-pick
helpers welded into your existing CLI workflow rather than a
full-screen TUI you enter and exit (`lazygit` wins for
session-y review work; `forgit` wins for in-line one-shot
operations inside a normal shell session). Pairs naturally
with `delta` (the recommended preview pager — colored
side-by-side hunk previews inside the `fzf` window),
`fzf-tab` (general fuzzy-completion of git refs), and any
existing alias setup (`forgit` aliases coexist with personal
`git` aliases without collision). The killer property is
**zero new mental model**: you already know `git add`,
`git checkout`, `git stash` — `forgit` just gives the
*selection* step a real picker.
