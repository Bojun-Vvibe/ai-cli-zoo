# halloy

> **A modern, IRCv3-first IRC client written in Rust on the
> Iced GUI toolkit** — single binary, no Electron, no
> background daemon, with first-class SASL/EXTERNAL, multi-server
> tabs, and per-server themes. Pinned to **2025.6**
> ([LICENSE](https://github.com/squidowl/halloy/blob/main/LICENSE),
> GPL-3.0-or-later).

Source: <https://github.com/squidowl/halloy>

## TL;DR

`halloy` is what you reach for when you still live on IRC in
2026 but want the polish of a modern chat app: native windowing
(no terminal multiplexer juggling), inline image previews,
URL/regex highlights, per-buffer notification rules, and a
config that is a single TOML file you can dotfile-version.
IRCv3 is the headline — `server-time`, `chghost`, `account-tag`,
`message-tags`, `extended-join`, plus SASL PLAIN / EXTERNAL
(client cert) for Libera.Chat / OFTC — so reconnects don't
shred channel context and bouncerless workflows actually work.
Themes are first-class (drop a `.toml` in `themes/`, hot-reload
on save), and the entire pane layout is splitable horizontally
and vertically per server, which is the feature long-time
weechat users miss in every other GUI client.

## Install

```bash
# Homebrew (macOS / Linux)
brew install halloy

# Flatpak (Linux)
flatpak install flathub org.squidowl.halloy

# Cargo from source (Rust >= 1.75)
cargo install --locked --git https://github.com/squidowl/halloy --tag 2025.6

# Verify
halloy --version    # halloy 2025.6
```

## Example usage

```bash
# 1. First launch — opens a config wizard, writes
#    ~/.config/halloy/config.toml on Linux,
#    ~/Library/Application Support/halloy/config.toml on macOS
halloy

# 2. Minimal config: connect to Libera.Chat with SASL PLAIN
cat >> ~/.config/halloy/config.toml <<'TOML'
[servers.libera]
nickname = "yourname"
server   = "irc.libera.chat"
port     = 6697
use_tls  = true
channels = ["#rust", "#archlinux"]

[servers.libera.sasl.plain]
username = "yourname"
password = "$env:LIBERA_PASS"     # read from $LIBERA_PASS at launch
TOML

# 3. SASL EXTERNAL with client cert (bouncerless, no password)
#    Generate a cert pair, upload pubkey to NickServ via CertFP, then:
cat >> ~/.config/halloy/config.toml <<'TOML'
[servers.oftc]
nickname    = "yourname"
server      = "irc.oftc.net"
port        = 6697
use_tls     = true
client_cert = "~/.config/halloy/oftc.pem"

[servers.oftc.sasl.external]
TOML

# 4. Per-server theme override + 24-hour timestamps
cat >> ~/.config/halloy/config.toml <<'TOML'
[buffer.timestamp]
format = "%H:%M:%S"

[servers.libera.theme]
name = "ferra"
TOML

# 5. Pop a vertical split for a busy channel without losing the
#    server pane:  Cmd/Ctrl+B then drag, or right-click the tab
#    → "Open in new pane (vertical)".
```

## Why this lives in the zoo

The IRC client space splits into "terminal-only and
script-heavy" (`weechat`, `irssi`) and "Electron with no
IRCv3" (most GUI clients shipped in the last decade). `halloy`
is the rare third option: a native-GUI Rust binary that takes
IRCv3 seriously, treats SASL EXTERNAL / CertFP as a first-class
auth method (so you don't need a bouncer just to survive a
disconnect with NickServ), and ships a config format you can
actually keep in a dotfiles repo. For users who want IRC
without either a terminal or a Chromium runtime, it is the
only credible option as of 2026.
