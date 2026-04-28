# television

- **Repo:** https://github.com/alexpasmantier/television
- **Version:** 0.15.6 (latest stable, 2025)
- **License:** MIT ([LICENSE](https://github.com/alexpasmantier/television/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install television` · `cargo install television` ·
  `pacman -S television` · prebuilt binaries on the GitHub release page ·
  binary name is `tv`

## What it does

`tv` is a general-purpose fuzzy finder TUI in the spirit of `fzf`, but
designed around named, configurable "channels" — recipes that pair a
data source with a preview command.

- Ships built-in channels for files, git log, git diff, processes,
  shell history, and environment variables; `tv files`, `tv git-log`,
  `tv env` etc. all open the same fuzzy picker against different
  sources.
- Channels are declarative TOML: a source command (anything that
  produces lines) plus a preview command that receives the highlighted
  line on stdin or as `{}`. Adding a channel for "all my k8s pods" or
  "every Cargo.toml in this monorepo" is a few lines of config.
- Renders previews in a side pane with syntax highlighting, image
  rendering (sixel / kitty graphics where supported), and live refresh
  when the highlighted entry changes.
- Cable-style channel composition: a channel can pipe into another
  channel, so you can fuzzy-pick a file, then fuzzy-pick a line in
  that file's git history, then preview the diff — without leaving
  the TUI.
- Standalone binary, no shell integration required, but ships ready-
  made shell hooks for bash / zsh / fish / nushell that bind common
  channels to keychords.

## When to use it

Reach for `tv` when `fzf` is what you'd normally use but you've found
yourself rebuilding the same `find ... | fzf --preview ...` incantation
across half a dozen aliases — `tv` lets you save those as named
channels and share them across machines via a config file. It's
particularly nice for repository-scoped pickers (a custom channel for
"all TypeScript files plus their import graph", or "every entry in the
incident log with the postmortem doc as the preview") that you want to
hand to a teammate as a one-line config addition rather than a shell
alias. The graphics support also makes it the better pick when previews
include images (icon files, screenshots in a docs tree).

## When NOT to use it

Skip it when you just need an ad-hoc one-shot picker — `fzf` is the
default for a reason, has the largest ecosystem of pre-baked
integrations, and is what every dotfiles repo and tool README assumes.
Skip it when your pickers must run on a constrained / minimal
environment where adding another binary isn't justified. Skip it for
strictly non-interactive pipelines (`tv` is a TUI; don't reach for it
inside a script that needs to run unattended). And skip it if you're
already invested in `skim` or another finder and your muscle memory is
tuned to its keybindings — the win from `tv` is the channel system, not
raw fuzzy-find performance.
