# senpai

> **A modern terminal IRC client built around IRCv3 capabilities — buffer playback (`chathistory`), away-notify, account-tag, server-time, message-tags, labeled-response, multi-prefix — so an IRC session on a modern bouncer (`soju`, `ZNC` with the right modules) feels closer to a Slack / Matrix client than to 1998-era irssi.**
> A small Go TUI by delthas (mirrored from
> [sr.ht/~delthas/senpai](https://sr.ht/~delthas/senpai/) to GitHub) that
> opens with a single `senpai` command, autoconnects to one or more
> configured networks, and renders each channel/query in a buffered window
> with vi-style navigation, completion (nicks / channels / commands), and
> a `:` command bar. Pinned to **v0.4.1**
> ([LICENSE](https://github.com/delthas/senpai/blob/master/LICENSE),
> ISC).

Source: <https://github.com/delthas/senpai> (mirror of <https://sr.ht/~delthas/senpai>)

## TL;DR

The traditional IRC TUI lineup — [`weechat`](https://weechat.org/),
`irssi`, [`halloy`](../halloy/) on the GUI side — predates the IRCv3
working group's modern capability set by years, and supports the new
caps either via plugins (weechat) or not at all (irssi without
patches). When you connect those clients to a modern bouncer like
[`soju`](https://soju.im/) you have to manually configure
`chathistory` polling, message-tag display, account-tag-based name
coloring, and labeled-response correlation — a long `.weechatrc` of
`/set` lines that no documentation tells you to write, and that
silently degrades to "looks like 1998 IRC" when any cap negotiation
fails.

`senpai` is "IRCv3 first" — every modern capability the bouncer
advertises is negotiated and used by default, and the UI is designed
around them. Scrollback comes from `chathistory` instead of from a
local log file, so a fresh `senpai` on a new laptop shows the same
backlog as the one running on your desktop. Away/return events
render inline with timestamps from `server-time` (so a 10-hour
backlog replay shows the actual times messages were sent, not the
times your client received them on connect). Notifications are
deduplicated across the bouncer-attached fleet via `bouncer-networks`
and `read-marker`, so a mention that fired on the desktop client
does not fire again on the laptop.

## Install

```bash
# Go (any platform with a Go 1.21+ toolchain)
go install codeberg.org/emersion/senpai/cmd/senpai@latest
# (the canonical install path for the senpai binary)

# Homebrew tap (community)
brew install senpai

# From source
git clone https://github.com/delthas/senpai
cd senpai && make && sudo make install

# verify
senpai -version
```

After install, drop a `~/.config/senpai/senpai.scfg`:

```scfg
address ircs://my-bouncer.example.org:6697
nickname myhandle
password-cmd pass show irc/my-bouncer
```

(Senpai uses [scfg](https://git.sr.ht/~emersion/scfg), an scfg-format
config — ~10 lines for a single-network setup, ~30 for a multi-network
bouncer.)

## Usage

```bash
# 1) Open the client; autoconnect to whatever is in senpai.scfg
senpai

# 2) Inside the TUI, talk to a channel
/join #archlinux
hi

# 3) Pull older history from the bouncer (no local log file required)
/topic            # show channel topic
/msg NickServ status
# Ctrl-P / Ctrl-N navigate buffers; Ctrl-A toggles the buffer list;
# `/` opens a search buffer over visible scrollback.

# 4) One-shot from a script (e.g. send "deploy started" to #ops)
senpai -config ~/.config/senpai/ops.scfg < <(echo "/msg #ops deploy started")
```

## Niche & tradeoffs

`senpai` lives in a narrow but well-defined slot: modern IRCv3 TUI
client paired with a modern bouncer. Distinct from:

- [`weechat`](https://weechat.org/) / `irssi` — the universal,
  scriptable, plugin-rich incumbents. Pick weechat when you want
  Lua/Python/Perl scripting, decades of community recipes, and
  XMPP / Matrix / Slack bridges in one process. Pick senpai when
  you want IRC-only with IRCv3 caps as the *default*, no plugin
  surface to configure, and a UI built for the modern protocol
  rather than retrofitted onto it.
- [`iamb`](../iamb/) — Matrix-protocol TUI in the vi-keymap
  family (which senpai shares stylistically). Use iamb when the
  homeserver is Matrix; senpai when the network is IRC. The two
  coexist on the same desktop.
- [`halloy`](../halloy/) — Rust GUI IRC client (iced toolkit),
  also IRCv3-first; pick halloy when you want a graphical
  client with a mouse-driven UI, senpai when the workflow is
  TUI-only and lives next to [`tmux`](https://tmux.github.io/),
  [`helix`](../helix/), or [`zellij`](../zellij/).
- [`tut`](../tut/) (Mastodon TUI), [`toot`](../toot/) (Mastodon
  CLI/TUI), [`circumflex`](../circumflex/) (Hacker News TUI) —
  same TUI-for-a-social-protocol family, different protocols.

The right pairing is **senpai + soju**: soju is the IRCv3 bouncer
written by the same broader contributor circle, and the two are
explicitly designed against each other's capability set. The
combined experience — `chathistory` backlog, `read-marker` cross-device
sync, `bouncer-networks` per-network attach — is the closest IRC has
come to "Slack-shaped, but it's IRC and you own the server."

Caveats — (1) IRC-only by design; no XMPP/Matrix/Slack bridges, no
plugin runtime. If you want one client for many protocols, stay on
weechat with its bridge plugins. (2) Pre-1.0 (`v0.4.x`) — the scfg
schema and a handful of keybindings have shifted between minor
releases; pin the version in your dotfiles and read the changelog
before upgrading. (3) Mouse support is intentionally minimal — the
keymap assumes a vi/tmux-shaped operator; the learning curve is real
for newcomers from Slack-style GUI clients. (4) The upstream is
sourcehut (sr.ht/~delthas/senpai); the GitHub repo is a mirror, so
issues and patches go via the sr.ht tracker / mailing list, not
GitHub PRs.
