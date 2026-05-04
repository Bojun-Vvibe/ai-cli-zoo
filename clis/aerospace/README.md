# aerospace

> **i3-like tiling window manager for macOS** — a single Swift
> binary plus a menu-bar agent that talks to the macOS
> Accessibility API (`AXUIElement`), arranges your real
> application windows into a BSP / horizontal / vertical /
> accordion tree, and exposes the entire workflow over a CLI
> (`aerospace workspace 3`, `aerospace move-node-to-workspace
> 4`, `aerospace focus left`) plus a plain-text TOML
> configuration — pinned to **v0.20.3-Beta** (commit
> [`6dde91b`](https://github.com/nikitabobko/AeroSpace/commit/6dde91ba43f62b407b2faf3739b837318266e077),
> [LICENSE.txt](https://github.com/nikitabobko/AeroSpace/blob/v0.20.3-Beta/LICENSE.txt),
> MIT).

Source: <https://github.com/nikitabobko/AeroSpace>

## TL;DR

macOS does not ship a real tiling window manager. The native
"Mission Control" + "Spaces" model is animation-heavy, slow to
switch, can't be scripted from a shell, and silently rewires
itself when you plug in a second monitor. The historical
options have been
[`yabai`](https://github.com/koekeishiya/yabai) (powerful but
requires partially disabling System Integrity Protection to
unlock its scripting addition, and breaks on every macOS point
release) and
[`Amethyst`](https://github.com/ianyh/Amethyst) (no SIP work,
but no real CLI surface and no i3-style workspace tree).

`aerospace` is the right shape: it implements its **own
emulation of virtual workspaces** instead of leaning on macOS
Spaces, so workspace switching is instant and animation-free,
and it works on a stock-SIP machine without kernel extensions,
private signing entitlements, or a daemon-restart dance after
each macOS update. The window tree is i3-style (split
horizontal / vertical, BSP-balanced, or accordion-tabbed), the
config is a single dotfile-friendly `~/.config/aerospace/aerospace.toml`,
and every interactive operation is also a one-line CLI command
that you can bind to a hotkey, call from a script, or wire into
[`skhd`](https://github.com/koekeishiya/skhd) /
[`raycast`](https://www.raycast.com/).

## Install

```bash
# Homebrew cask (preferred — autoupdates, auto-strips
# com.apple.quarantine, registers as a Login Item)
brew install --cask nikitabobko/tap/aerospace

# Verify
aerospace --version          # → 0.20.3-Beta
```

The cask installs the GUI agent at `/Applications/AeroSpace.app`
plus the `aerospace` CLI at `/opt/homebrew/bin/aerospace` (the
CLI is just an IPC client — the GUI agent must be running for
any command to do anything).

`aerospace` runs on macOS 13 (Ventura), 14 (Sonoma), 15
(Sequoia), and 26 (Tahoe). It is **not notarized by Apple** —
the Homebrew cask handles the quarantine attribute for you, but
the project documents this explicitly and the upstream README
explains the reasoning.

## Day one

```bash
# 1. Generate a default config (one-shot, idempotent)
aerospace config --generate
# → writes ~/.config/aerospace/aerospace.toml

# 2. Bind hotkeys inline in the TOML — example excerpt:
cat >> ~/.config/aerospace/aerospace.toml <<'TOML'
[mode.main.binding]
alt-h = 'focus left'
alt-j = 'focus down'
alt-k = 'focus up'
alt-l = 'focus right'

alt-1 = 'workspace 1'
alt-2 = 'workspace 2'
alt-3 = 'workspace 3'

alt-shift-1 = 'move-node-to-workspace 1'
alt-shift-2 = 'move-node-to-workspace 2'

alt-slash = 'layout tiles horizontal vertical'
alt-comma = 'layout accordion horizontal vertical'
TOML

# 3. Reload without restarting the agent
aerospace reload-config

# 4. Drive workspaces from anywhere — works in scripts, tmux,
#    a Stream Deck, a launchd job, etc.
aerospace workspace 2
aerospace list-windows --workspace focused --json
aerospace move-node-to-monitor next
```

## Why pin to v0.20.3-Beta

- The **codesign certificate was rotated** in this release;
  earlier 0.20.x builds will silently lose accessibility
  permission on a fresh macOS install.
- Two long-standing floating-window bugs are fixed:
  - floating windows could appear off-screen when moved between
    monitors of different geometry,
  - floating windows could "stick" in a corner after a
    workspace switch.
- Inherits the 0.20.0 feature set: `persistent-workspaces`
  config option, `--json` / `--count` flags on `list-modes`,
  i3-ordered menubar style, `swap` and `focus dfs-next` /
  `focus dfs-prev` commands, and the `on-mode-changed`
  callback.

The project is pre-1.0 — upstream is explicit that breaking
changes happen between minor versions. If you script against
`aerospace`, pin the binary version (the cask supports this via
`brew install --cask nikitabobko/tap/aerospace@<version>` once
the maintainer cuts a tap entry, otherwise pin via your dotfiles
repo).

## When NOT to reach for it

- **You want a ricing-friendly WM with window borders, gaps
  animations, transparency, blur, etc.** Upstream is explicit:
  "ricing is a non-value." There are gaps, and a few callback
  hooks for status bars (sketchybar, Übersicht). That is the
  ceiling.
- **You need to keep using macOS-native Spaces / Mission
  Control gestures.** `aerospace` ignores native Spaces
  entirely; mixing the two paradigms is a recipe for confusion.
- **You want a GUI configurator.** There will never be one —
  the config is a TOML file, full stop.
- **You need full multi-display "follow the mouse" focus
  semantics on a setup with mirrored or rotated displays.**
  `aerospace` follows i3 monitor semantics; if your monitor
  arrangement in *System Settings → Displays* is wrong (e.g.
  monitors visually overlapping), `aerospace` will inherit the
  confusion. Fix the arrangement first.
- **You want sticky/always-on-top windows or a dynamic TWM
  paradigm** (à la dwm/xmonad). Both are explicitly post-1.0
  roadmap items, not present today.

If those exclusions don't apply, this is the macOS tiling WM
that gets out of your way and stays out of it.
