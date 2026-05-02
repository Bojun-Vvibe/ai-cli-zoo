# tut

> **Mastodon TUI client** — read your home / local / federated
> timelines, post, boost, favourite, follow, DM, and switch
> between multiple accounts entirely from the keyboard, in a
> three-pane terminal layout that loads instantly and stays out
> of the browser.
> Pinned to **2.0.1**
> ([LICENSE](https://github.com/RasmusLindroth/tut/blob/master/LICENSE),
> MIT).

Source: <https://github.com/RasmusLindroth/tut>

## TL;DR

`tut` is a Go TUI built on `tcell` that talks to any Mastodon
(or Pleroma / Akkoma / GoToSocial — any
Mastodon-compatible-API) instance over its REST + WebSocket
streaming API. First launch opens a browser tab for OAuth, drops
the token under `~/.config/tut/`, and from then on everything is
keyboard-driven: `j`/`k` to scroll the timeline, `Enter` to open a
toot's thread + media, `c` to compose, `b` to boost, `f` to
favourite, `t` to start a thread, `T` to switch timeline (home /
local / federated / notifications / lists / tags / bookmarks),
`g` then a number to switch accounts. Configuration is one TOML
file (`~/.config/tut/config.toml`) with key-rebind tables, theme
hex codes, and per-instance `[[account]]` blocks for multi-account
use. No browser, no JS, no Electron — federated social fits in a
`tmux` pane.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install tut

# Arch (AUR)
yay -S tut

# Nix
nix-shell -p tut

# Go install (latest)
go install github.com/RasmusLindroth/tut@latest

# Pre-built binary (Linux amd64 static)
curl -L -o /usr/local/bin/tut \
  https://github.com/RasmusLindroth/tut/releases/download/2.0.1/tut-amd64-static
chmod +x /usr/local/bin/tut

# verify
tut --version    # 2.0.1
```

## Examples

```bash
# Launch — opens OAuth in browser on first run
tut

# Launch with a non-default config dir (useful for per-account testing)
tut --config-dir ~/.config/tut-alt

# Compose a single toot from the shell, no TUI (useful in scripts)
tut new "shipped a thing"
```

## Niche / category

Social-media TUI — Mastodon-shaped specifically. Not a generic
ActivityPub browser; the keymap, the timeline taxonomy, and the
compose flow are tuned for Mastodon-API instances.

## When to use

- You already use `tmux` / `zellij` and want the federated
  timeline in a pane next to your editor and shell, not in
  another window of another browser.
- You're on a low-bandwidth or screen-reader-friendly setup where
  the official Mastodon web UI's JS-heavy timeline is a tax.
- You run multiple Mastodon accounts (work / personal / fediverse
  alt) and want one config file with `[[account]]` blocks instead
  of three browser profiles.
- You want a Mastodon client that respects shell muscle memory
  (vim-style nav, `:` ex-commands for quick actions).

## When NOT to use

- You're on Bluesky / Twitter / Threads — those are not
  Mastodon-API; `tut` cannot talk to them. (For Bluesky, see
  `bsky-cli` / `tuiwitter`-style alternatives.)
- You need polls that are interactive in the same UI — `tut`
  reads polls but the editor for creating multi-option polls is
  limited; compose from the web UI then engage from `tut`.
- You're on Windows native — supported via the Go cross-compile
  but the terminal experience under `cmd.exe` is rough; use WSL.
- You want push notifications outside the running session — `tut`
  is foreground-only; pair with a separate notifier (Mastodon's
  email/web push) for "background" alerts.

## Related

- [`toot`](../toot/) — Mastodon CLI + TUI in Python (more
  scriptable from shell — `toot post "..."`, `toot timeline` —
  pick `toot` when one-shot CLI commands matter, pick `tut` when
  the long-lived TUI matters)
- [`circumflex`](../circumflex/) — Hacker News in the same
  three-pane shape (orthogonal — pair them in adjacent `tmux`
  panes)
- [`tuifeed`](../tuifeed/), [`newsboat`](../newsboat/) — RSS
  readers (pre-Mastodon way to follow the same accounts via
  their RSS feeds, no auth)
