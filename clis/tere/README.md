# tere

- **Repo:** https://github.com/mgunyho/tere
- **Version:** v1.6.0 (latest tagged release, 2024-09)
- **License:** EUPL-1.2 ([LICENSE](https://github.com/mgunyho/tere/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install tere` · `cargo install tere` · `pacman -S tere` · `nix-env -i tere` · `scoop install tere` · prebuilt binaries on the GitHub releases page

## What it does

`tere` ("hello" in Estonian) is a deliberately tiny TUI whose sole
purpose is to replace the `cd ↵ ls ↵ cd ↵ ls` loop that every shell
session devolves into. You launch `tere`, you see the directory
listing in a one-pane TUI, you press `j`/`k`/arrows to move the cursor
or — and this is the whole point — you just *start typing the name of
the folder you want* and tere incrementally narrows the listing using
gap-search ("dt" matches DeskTop and DocumenTs). When exactly one
folder matches, tere autocd's into it after a configurable timeout, so
a fast typist can fall multiple levels deep with about as many
keystrokes as the folder names share unique prefixes. Press Esc and
tere prints the final directory to stdout and exits — it cannot change
your shell's working directory itself (no subprocess can), so you wrap
it in a one-line shell function (`tere() { local r=$(command tere
"$@"); [ -n "$r" ] && cd -- "$r"; }`) that captures stdout and runs
`cd`. Beyond navigation and search, the feature surface is
intentionally tiny: no file management (no rename, no delete, no
copy), no preview pane, no plugin system, no theming engine. What it
does have: smart-case search (lowercase = case-insensitive, has-
uppercase = case-sensitive), filter mode (`--filter-search` hides non-
matching items instead of just dimming them), configurable keybindings
via `--map`, history of visited folders saved as JSON for
`--smart-completion`-style hints, and shell setup snippets in the README
for bash / zsh / fish / xonsh / nushell / PowerShell / cmd.

## When to pick it / when not to

Pick `tere` when 90% of your file-system "exploration" is actually
"navigate to a known folder I half-remember the name of, then cd
there" — that's the case for most developer terminals, and tere
collapses it to "type tere, type a few letters, hit Esc". The win
over plain `cd` + tab completion is gap-search ("typ srcapi" finds
`src/api/...` even when the path has hyphens, dots, and nested dirs
you didn't remember); the win over [`zoxide`](../zoxide/) is that you
don't need to have visited a folder before to jump there — tere is
exploration, zoxide is recall, and they pair perfectly (zoxide for
"cd to that project I worked on last week", tere for "now navigate
into a subfolder of it I've never opened"). Pick it specifically over
fuller file managers like [`yazi`](../yazi/) /
[`xplr`](../xplr/) / [`broot`](../broot/) / [`nnn`](https://github.com/jarun/nnn)
when you want zero learning curve, zero config, a single static
binary, and a single job done extremely well. Pair it with
[`fzf`](https://github.com/junegunn/fzf) (for file picking inside a
folder), [`fd`](../fd/) (for cross-tree search), and a normal text
editor — that combo covers most of what big file managers do, with no
mode-switching cost.

Skip it the moment you need to *do* something with files (rename,
delete, copy, preview, archive, chmod) — tere refuses to grow those
features, and you want [`yazi`](../yazi/) / [`xplr`](../xplr/) /
[`broot`](../broot/) / `mc` instead. Skip it when your bottleneck is
not knowing *which* folder you want — that's a [`zoxide`](../zoxide/)
or `fzf | xargs cd` problem, not a tere problem. Skip it on Windows
PowerShell if you're not willing to copy-paste the wrapper from the
README (the wrapper is required on every shell — there's no way around
it given how Unix process working directories work). And note the
EUPL-1.2 license is unusual for CLI tooling — it's an OSI-approved
copyleft license but with weak-copyleft semantics oriented around
European law; for personal use this changes nothing, but if you're
distributing tere as part of a commercial product, have your legal
team look at it the way they would for LGPL / MPL.

## Example invocations

```bash
# Just browse — type letters to filter, Enter to descend, Esc to "cd here & exit"
tere

# Bash / Zsh wrapper (put this in .bashrc / .zshrc)
tere() {
  local result=$(command tere "$@")
  [ -n "$result" ] && cd -- "$result"
}

# Fish wrapper (put this in config.fish)
function tere
  set --local result (command tere $argv)
  [ -n "$result" ] && cd -- "$result"
end

# Filter mode: hide non-matching folders instead of just greying them out
tere --filter-search

# Match files too (default is folders only, since tere can't open files)
tere --files=match

# Force case-sensitive search
tere --case-sensitive

# Disable gap search — only consecutive substrings match
tere --normal-search

# Enable mouse navigation (off by default)
tere --mouse=on

# Bind Ctrl-q to quit instead of Esc
tere --map ctrl-q:Exit

# Start in a specific directory (rather than $PWD)
tere ~/Projects
```
