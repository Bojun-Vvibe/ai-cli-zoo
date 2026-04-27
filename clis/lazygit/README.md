# lazygit

> **Git-as-a-keyboard-game** — a single Go binary that opens a
> five-panel ncurses-style TUI (status / files / branches /
> commits / stash) on top of any git repo, where every common
> git operation (stage hunk, amend, interactive rebase, cherry-
> pick, conflict-resolve, bisect, push, force-push-with-lease,
> submodule update) is one or two keystrokes — and the
> interactive-rebase TODO list in particular is editable with
> `j` / `k` / `s` (squash) / `f` (fixup) / `d` (drop) / `e`
> (edit) instead of "set `EDITOR=vim` and pray". Pinned to
> **v0.61.1** (commit
> `b4f79851dba1337e72b6b2f0180239c9d59d4f8f`,
> [LICENSE](https://github.com/jesseduffield/lazygit/blob/master/LICENSE),
> MIT).

Source: <https://github.com/jesseduffield/lazygit>

## TL;DR

`lazygit` is what you reach for when the cost of typing
`git status && git add -p && git commit -v --amend` for the
twentieth time today is finally higher than the cost of
learning a new keymap. `cd` into any repo, type `lazygit`, and
you get a live-updating TUI with five focusable panels: status
(repo summary + recent commands), files (working tree + index
with stage / unstage / discard per-file *or* per-hunk *or* per-
line), branches (local + remote with checkout / merge / rebase /
fast-forward / delete), commits (log with squash / fixup /
reword / drop / pick / edit / cherry-pick / revert / amend /
diff against HEAD), and stash (push / pop / apply / drop). The
killer feature is **interactive rebase as a live TUI**: select a
range in the commits panel, press `e` to start an interactive
rebase, then move commits with `Ctrl+J` / `Ctrl+K`, fixup with
`f`, squash with `s`, drop with `d`, all without ever opening
the rebase TODO file. Conflict resolution gets the same
treatment — a conflicted file shows ours / theirs / both as
selectable hunks, `space` picks one, `c` continues the rebase.
The whole thing is one statically-linked Go binary with a
`config.yml` for keybind overrides, a custom-commands DSL for
project-specific actions, and full mouse support if you turn it
on.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lazygit

# Go
go install github.com/jesseduffield/lazygit@latest

# Linux package managers
# Arch: pacman -S lazygit
# Debian / Ubuntu (PPA): see README — not in default apt
# Fedora: dnf copr enable atim/lazygit -y && dnf install lazygit

# from a release tarball (any OS)
LAZYGIT_VERSION=$(curl -s "https://api.github.com/repos/jesseduffield/lazygit/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
curl -Lo lazygit.tar.gz "https://github.com/jesseduffield/lazygit/releases/latest/download/lazygit_${LAZYGIT_VERSION}_Darwin_arm64.tar.gz"
tar xf lazygit.tar.gz lazygit
sudo install lazygit /usr/local/bin/

# verify
lazygit --version    # date=... version=0.61.1 ...
```

Zero git config required — `lazygit` shells out to your existing
`git` binary, respects `~/.gitconfig`, and writes its own state
under `~/Library/Application Support/lazygit/` (macOS) /
`~/.config/lazygit/` (Linux) / `%APPDATA%\lazygit\` (Windows).
First-run prompt offers to set up a `git` alias (`git config
--global alias.lg lazygit`) so `git lg` opens it from anywhere.

## License

MIT — see [LICENSE](https://github.com/jesseduffield/lazygit/blob/master/LICENSE).
Permissive, no attribution required for binaries; redistribute
freely.

## Hot keybinds

These are the ones that pay back the learning cost in the
first hour:

- `?` — context-sensitive help (every panel has its own keymap
  page; the help screen is the single most-used onboarding tool)
- `[` / `]` — cycle focused panel left / right (Tab / Shift-Tab
  also work)
- **Files panel:** `space` stage / unstage file, `enter` open
  the per-hunk / per-line staging view, `d` discard, `c` commit
  staged, `C` commit with editor (`EDITOR`), `A` amend last
  commit, `e` open file in `EDITOR`
- **Branches panel:** `space` checkout, `n` new branch, `M`
  merge, `r` rebase, `f` fast-forward, `d` delete, `R` rename,
  `o` create PR (opens browser via `gh` if installed)
- **Commits panel:** `s` squash into commit below, `f` fixup
  into commit below, `r` reword, `e` start interactive rebase
  with this commit at top, `d` drop, `p` pick (during rebase),
  `c` cherry-pick (mark with `C` first), `R` revert, `t` tag,
  `Ctrl+J` / `Ctrl+K` move commit down / up during rebase,
  `m` continue / abort rebase or merge
- **Stash panel:** `space` apply, `g` pop, `d` drop, `n` new
  stash from working tree, `r` rename
- **Global:** `:` open command-log + execute raw git command,
  `P` push, `p` pull, `Shift+P` force-push-with-lease, `Q`
  quit, `Esc` cancel current modal

## Why use it

Three things `lazygit` does that the bare `git` CLI does not,
that pay back the learning cost permanently:

1. **Per-hunk + per-line staging without `git add -p`'s
   modal-keystroke prompts.** In the files panel, `enter` on a
   modified file opens a side-by-side diff where `space` stages
   the hunk under the cursor, `Tab` switches to per-line mode,
   and arrow keys move the selection. Building a clean commit
   from a messy working tree drops from "five minutes of `git
   add -p` and `e` and `s` and `j` and `n`" to "thirty seconds
   of arrow keys + space".
2. **Interactive rebase as a live TUI.** Selecting a commit and
   pressing `e` turns the commits panel into an editable
   rebase TODO. `s` squash, `f` fixup, `d` drop, `r` reword,
   `Ctrl+J` / `Ctrl+K` reorder — all live, all visible, all
   undoable with `z` (the recovery command-log). The "rebase
   broke and I am stuck in a detached HEAD with `vim` open on
   a `.git/rebase-merge/git-rebase-todo` file" failure mode
   stops happening.
3. **Conflict resolution with picker UI instead of conflict
   markers in your editor.** A conflicted file in the files
   panel opens a three-way diff (ours / theirs / both); `space`
   picks the highlighted hunk, `b` keeps both, `c` continues
   the rebase / merge once all conflicts are resolved. The
   `<<<<<<<` / `=======` / `>>>>>>>` editor dance disappears.

For an LLM-CLI workflow that produces frequent small commits
that need squashing / reordering / amending before being pushed
as a clean PR, `lazygit` collapses the "clean up the history
before push" step from minutes to seconds.

## Vs Already Cataloged

- **Vs [`gitui`](../gitui/):** the closest peer — both are
  TUI git clients. `gitui` is Rust, faster on huge repos
  (`linux.git`-class), has a smaller feature surface (no
  custom-commands DSL, no built-in PR-opening, more spartan
  rebase UI). Pick `gitui` for raw speed on monorepos, pick
  `lazygit` for the richer interactive-rebase + custom-commands
  + GitHub-integration story.
- **Vs [`jj`](../jj/):** different layer entirely. `jj` replaces
  git's commit model (every working-tree edit is a commit, the
  history is a first-class operation log). `lazygit` is a UI on
  top of *unmodified* git. Use `lazygit` when the team is on
  git and you want a faster keymap; use `jj` when you want to
  change the underlying VCS semantics.
- **Vs [`delta`](../delta/):** orthogonal — `delta` is a *diff
  pager* (replaces `less` for `git diff` / `git log -p`),
  `lazygit` is a *git client*. They compose: set
  `git config --global core.pager delta` and `lazygit`'s
  external-diff modals (press `D` on a commit) render through
  delta automatically.
- **Vs [`gh`](https://cli.github.com/) / [`glab`](https://gitlab.com/gitlab-org/cli):**
  those are GitHub / GitLab API clients (PRs, issues, releases,
  workflows). `lazygit` is local-git only — but its `o` keybind
  on a branch / commit shells out to `gh pr create` if `gh` is
  on `$PATH`, so they compose for the "finished a feature
  branch, want to open a PR without leaving the terminal" flow.

## Caveats

- **TUI assumes a true-color terminal** with at least 80 columns
  and 24 rows; under ~100 cols the five-panel layout stacks
  vertically and the per-hunk staging view gets cramped. Works
  fine in `tmux`, `screen`, `iTerm2`, `kitty`, `wezterm`,
  `alacritty`, modern Windows Terminal — break in older
  Windows console host (`cmd.exe`).
- **Custom-commands DSL is YAML** in `~/.config/lazygit/config.yml`
  under `customCommands:` — easy to add a project-specific
  keystroke (e.g. `r` to reset to `origin/main` with a confirm
  prompt) but the `prompts:` syntax has its own quirks; read
  the [docs/custom_commands.md](https://github.com/jesseduffield/lazygit/blob/master/docs/Custom_Commands.md)
  before hand-rolling complex ones.
- **`Shift+P` (force-push-with-lease) is one keystroke and the
  confirm modal is small** — easy to muscle-memory through
  while reviewing a different terminal pane. Pair with a CI
  `branch-protection` rule that blocks force-push to `main` /
  `master` if you share repos with a team.
- **Recovery via `z` (command-log undo)** only undoes operations
  `lazygit` performed itself this session — a `git reset
  --hard` typed in another terminal pane is *not* in the log.
  Treat it as a session-local convenience, not a substitute for
  `git reflog`.
- **No first-class git-LFS UI** — LFS pointers show as normal
  files; clean / smudge filters run via the underlying `git`
  binary as expected, but LFS-specific operations (`git lfs
  pull`, `git lfs migrate`) are not in the keymap (use the `:`
  command palette).
