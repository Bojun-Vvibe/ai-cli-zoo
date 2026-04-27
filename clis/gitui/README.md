# gitui

> **Blazingly fast terminal-UI for git, written in Rust** — a
> single static binary that renders status, stage, diff, log,
> branches, stashes, and remotes as a navigable TUI with
> instant key-driven actions. Pinned to **v0.28.1**
> ([LICENSE-MIT](https://github.com/extrawurst/gitui/blob/master/LICENSE-MIT),
> MIT).

Source: <https://github.com/extrawurst/gitui>

## TL;DR

`gitui` is what you reach for when `git status`, `git add -p`,
`git diff --cached`, `git log --oneline --graph`, and
`git stash list` would all be the next five commands you type.
A single ~10 MB Rust binary, no daemon, no Node, no Python —
just opens in the current repo and shows everything in one
keyboard-driven TUI. Hunk-level staging is a single keystroke
(`s` to stage, `u` to unstage, `enter` to drill in), the log
view supports inline diff inspection and interactive rebase, and
the whole thing redraws fast enough to feel like a native app
even on a 50k-commit repo.

## Install

```bash
# Homebrew (macOS / Linux)
brew install gitui

# Linux package managers
# Arch:   pacman -S gitui
# Nix:    nix-env -iA nixpkgs.gitui

# Cargo (any OS with a Rust toolchain)
cargo install gitui --locked

# from a release tarball
curl -Lo gitui.tar.gz "https://github.com/extrawurst/gitui/releases/download/v0.28.1/gitui-mac-aarch64.tar.gz"
tar xf gitui.tar.gz && sudo install gitui /usr/local/bin/

# verify
gitui --version    # gitui 0.28.1

# launch (just `cd` into a repo first)
gitui
```

## License

MIT — see
[LICENSE-MIT](https://github.com/extrawurst/gitui/blob/master/LICENSE-MIT).
Permissive, no attribution required for binaries.

## Niche It Fills

**`git` as a TUI, not a flag salad.** The `git` CLI is fine when
you know exactly which subcommand and which flags, but the
moment you're in "review the diff, stage these three hunks but
not those two, write a message, then look at the log graph"
mode, you alternate `git status`, `git add -p`, `git diff
--cached`, `git commit`, `git log` for ten minutes. `gitui`
collapses that loop into a four-pane keyboard view with hunk-
level staging on a single keystroke.

## Why it pairs with coding agents

For agent / LLM workflows where the model is iterating on a
repo (edit, build, test, fix, repeat), keeping `gitui` open in
a side pane gives the human reviewer constant-time read on
"what did the agent actually change since my last sync" without
typing `git diff` between every iteration. Hunk-level unstage
(`u`) is the fastest way to reject a single bad change the
agent made while keeping the rest of the diff.

## Vs Already Cataloged

- **Vs [`lazygit`](../lazygit/):** sibling-in-spirit. `lazygit`
  is Go + `gocui`, `gitui` is Rust + `tui-rs`. Same five-pane
  shape, same keyboard-first muscle memory. `gitui` tends to
  feel snappier on huge repos; `lazygit` has a slightly larger
  feature surface (custom commands, more menus). Most users
  pick one and stick with it.
- **Vs [`delta`](../delta/):** `delta` is a *pager* for `git
  diff` output — beautiful, but you still type the `git`
  commands yourself. `gitui` is the TUI shell; you can pipe
  through `delta` for the diff pane (`[diff] tool = delta` in
  config) to get both.

## Caveats

- **No interactive rebase editor.** `gitui` supports kicking
  off a rebase, but the actual todo-list edit drops you into
  `$EDITOR`. It's a TUI for git, not a replacement for
  `rebase -i`'s editor flow.
- **Single-repo at a time.** No multi-repo dashboard view; one
  process, one working directory.
