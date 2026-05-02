# byobu

> **Opinionated wrapper over `tmux` (or `screen`)** that ships a
> curated keybinding set, a status-bar full of system telemetry
> (load, RAM, IP, battery, updates, uptime, reboot-required), and
> session-persistence-by-default — so a single `byobu` invocation
> gives you a multiplex shell that survives SSH drops without
> writing one line of `tmux.conf`. Pinned to **v6.14** (released
> 2026-02-15,
> [LICENSE](https://github.com/dustinkirkland/byobu/blob/master/COPYING),
> GPL-3.0).

Source: <https://github.com/dustinkirkland/byobu>

## TL;DR

`byobu` is what you reach for when you want the *benefits* of
`tmux` (detach/reattach, splits, named sessions surviving
disconnects) without spending a Saturday on `tmux.conf`. On a
fresh box: `apt install byobu && byobu` and you immediately get:

- **F-key UX** — `F2` new window, `F3`/`F4` prev/next window,
  `F5` reload, `F6` detach, `F7` enter copy/scrollback mode,
  `F8` rename window, `F9` config menu, `Shift-F2` horizontal
  split, `Ctrl-F2` vertical split. No prefix-key gymnastics
  (the `Ctrl-b`/`Ctrl-a` prefix still works for muscle memory,
  but you don't *have* to use it).
- **A status bar that's actually useful by default** — load
  average, memory, disk, IP address, who's logged in, pending
  package updates, "reboot required" flag, uptime, current
  hostname, custom user scripts in `~/.byobu/bin/`. Toggle each
  via the `F9` menu; no scripting required.
- **Session-on-login.** `byobu-enable` adds it to your shell rc
  so every interactive login lands you inside a persistent
  byobu session — close your laptop, reconnect from another
  machine, your shell history and running jobs are still there.
- **Same binary, two backends.** `byobu-tmux` (default modern
  backend) and `byobu-screen` (legacy GNU `screen` backend).
  `byobu-select-backend` switches; the keybindings stay the
  same, so muscle memory survives the migration off `screen`.

## How it compares vs alternatives in this zoo

- vs raw `tmux` — tmux is the engine; byobu is a
  pre-built status bar + keymap + session manager on top of it.
  Pick raw tmux when you want full control of `tmux.conf` and
  you'll learn the prefix-key vocabulary anyway. Pick byobu when
  you SSH into 50 different boxes and want them all to behave
  the same with zero per-box setup, *and* you want a status bar
  that tells you "this server has 47 pending updates and needs a
  reboot" without writing it.
- vs [`zellij`](../zellij/) — zellij is the modern Rust
  multiplexer with a discoverable mode-bar, layout files
  (KDL-defined), and a WASM plugin system. byobu is the
  conservative shell-script-and-awk wrapper that's been in the
  Ubuntu archive since 2009 and is preinstalled on Ubuntu Server.
  Pick zellij for greenfield laptops; pick byobu when "it's
  already there on every server I touch" matters more than the
  newest feature set.
- vs [`tmuxp`](../tmuxp/) — tmuxp is a *session-definition* tool
  (YAML/JSON files describing windows, panes, and startup
  commands). byobu is a *runtime UX* tool. They compose: byobu
  can be your default backend, with tmuxp recipes loading
  project-specific layouts inside a byobu session.
- vs `screen` directly — same era, same niche, but byobu's
  default keybindings + status bar make `screen` actually
  pleasant. If you must use `screen` (locked-down hosts, no
  tmux available), byobu's `screen` backend is the most
  ergonomic way to do it.

## Install

```bash
# Linux package managers (preferred — it's preinstalled on Ubuntu Server)
# Debian / Ubuntu: apt install byobu
# Fedora: dnf install byobu
# Arch (AUR): yay -S byobu
# Alpine: apk add byobu

# macOS
brew install byobu     # uses tmux backend by default

# from source (any UNIX with bash + tmux>=1.5 or screen)
git clone https://github.com/dustinkirkland/byobu.git
cd byobu && ./autogen.sh && ./configure --prefix="$HOME/.local"
make && make install

# verify
byobu --version | head -1
```

## Examples

```bash
# first run: just type byobu — it creates or attaches to a session
byobu

# pick a backend explicitly (tmux is the default and recommended)
byobu-select-backend tmux

# auto-launch byobu on every interactive login
byobu-enable
# undo:
byobu-disable

# inside a byobu session — F-keys, no prefix needed:
#   F2          new window
#   F3 / F4     prev / next window
#   F6          detach (your jobs keep running)
#   Shift-F2    split horizontally
#   Ctrl-F2     split vertically
#   F7          enter scrollback / copy mode
#   F9          status-bar config menu

# reattach from anywhere
ssh user@server -t byobu      # SSH straight into your persistent session

# add a custom status-bar widget (runs every N seconds)
mkdir -p ~/.byobu/bin
cat > ~/.byobu/bin/60_kubectx <<'EOF'
#!/bin/sh
kubectl config current-context 2>/dev/null
EOF
chmod +x ~/.byobu/bin/60_kubectx
# F9 -> "Toggle status notifications" -> enable "kubectx"
```

## When NOT to reach for byobu

- You want to live inside a hand-tuned `tmux.conf` with custom
  prefix and bindings — byobu's whole pitch is *not* doing that.
  Use raw `tmux` or [`zellij`](../zellij/).
- You're on Windows-native (no WSL) — byobu is POSIX-only. Use
  [`wezterm`](../wezterm/) or Windows Terminal multiplexing
  instead.
- You only ever connect to one machine and never get
  disconnected — the persistence + cross-host-uniformity story
  doesn't apply, and the status-bar overhead is wasted.
