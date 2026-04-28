# gitu

- **Repo:** https://github.com/altsem/gitu
- **Version:** v0.41.0 (latest stable, 2026-03)
- **License:** MIT ([LICENSE](https://github.com/altsem/gitu/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install gitu` · `cargo install --locked gitu` · `nix run nixpkgs#gitu` · static binary releases on the GitHub release page · `pacman -S gitu` (Arch)

## What it does

`gitu` is a terminal git porcelain modeled directly on Emacs's `magit`.
Run it inside a repository and it opens a single full-screen TUI that
shows, top to bottom: the current branch / HEAD / upstream summary,
untracked files, unstaged changes (with hunk-level diffs expanded
inline), staged changes, recent commits, and the current stash list.
The point is keyboard navigation: every section can be folded with
`Tab`, individual hunks (and individual lines within a hunk) can be
staged or unstaged with `s` / `u`, and the same key on a file in the
"unstaged" section stages the whole file. `c` opens the commit menu
(`c` again for a normal commit, `a` for `--amend`, `e` for `--amend
--no-edit`, `f` for fixup, `w` for reword), `b` opens the branch menu
(checkout, create, rename, delete), `l` opens the log menu (current
branch, all branches, file history), `P` pushes (`p` pulls, capital `P`
because push is the destructive direction), `F` fetches, `r` rebases
(`i` for interactive, `c` for continue, `a` for abort, `s` for skip),
`y` copies the SHA of the cursor commit, `Enter` opens the commit at
the cursor as a full-screen diff. Crucially, *every* destructive action
opens a confirmation prompt that shows you the exact `git` command about
to run — so `gitu` doubles as a "what does the equivalent CLI command
look like" teacher when you need to do the same thing later from a
script. Configuration lives in `~/.config/gitu/config.toml`: keybindings,
theme, the external diff / merge tool, the pager, and which sections
appear in the status view are all overridable. The TUI is built on
`ratatui` and the binary is a single static Rust executable (~5 MB)
with no runtime dependency beyond `git` itself.

## When to pick it / when not to

Pick `gitu` when you live in the terminal and want hunk- and line-level
staging without leaving it — the magit `s` / `u` model on individual
diff lines is the feature `git add -p` was always trying to be, and it
is what makes "split this messy WIP buffer into three clean, reviewable
commits" a 60-second job instead of a 10-minute paper cut. Pair with
[`lazygit`](../lazygit/) only as comparison — they cover overlapping
ground; `gitu` follows the magit conventions more strictly (transient
menus, popup confirmations that show the underlying command, more
keyboard-only workflow), `lazygit` is more mouse-friendly and ships a
broader set of built-in panels. Pair with [`gh-dash`](../gh-dash/) for
PR-side review (`gitu` is local-only — branches, commits, hunks; PRs
and issues belong elsewhere), with [`delta`](../delta/) as the pager
inside `gitu` for syntax-highlighted diffs (configure in `config.toml`),
with [`difftastic`](../difftastic/) as the diff driver when you want
syntax-aware structural diffs in the staging view, and with
[`jj`](../jj/) when you want to leave git's mental model behind
entirely instead of getting a better porcelain on top of it.

Skip it if you do not already know git's primitives — `gitu` exposes
them, it does not hide them, so a confusing `rebase --onto` mid-conflict
is still a confusing `rebase --onto` mid-conflict, just with nicer
arrow-key navigation through the conflict files. Skip it if you live in
Emacs and `magit` already works for you — `gitu` exists *because*
loading magit outside Emacs is awkward; if you are already inside the
parent ship you do not need the lifeboat. Skip it for tiny one-shot
operations (`git commit -am 'wip' && git push` is faster from the
command line than launching a TUI and pressing four keys). Skip it
for repository hosting features (PR creation, code review comments,
CI status, releases) — that is `gh` / `glab` / `gh-dash` territory;
`gitu` is strictly a local-repo porcelain.

## Example invocations

```bash
# Open the TUI on the current repo (the only "command" you usually run)
gitu

# Open it on a specific repo path without cd-ing first
gitu -C ~/code/myproject

# Run a single git command through gitu's confirmation UI (rare; mostly used
# inside the TUI, but exposed for scripting muscle memory)
gitu --version

# Inside the TUI, the muscle-memory keys you use 95% of the time:
#   Tab         fold / unfold the section under the cursor
#   s           stage the file / hunk / line under the cursor
#   u           unstage the file / hunk / line under the cursor
#   k           discard (with confirmation popup showing the git command)
#   c c         commit
#   c a         commit --amend
#   c e         commit --amend --no-edit
#   c f         commit --fixup=<sha at cursor>
#   b b         checkout branch
#   b c         create branch from HEAD
#   l l         log current branch
#   l a         log all branches
#   r i         rebase --interactive onto a chosen commit
#   F           fetch (lowercase p pulls; capital P pushes)
#   y           copy the SHA / branch name / file path under the cursor
#   ?           open the help / command reference
#   q           quit

# Configure the pager and theme by editing the config file
$EDITOR ~/.config/gitu/config.toml
```
