# flameshot

> **Powerful, scriptable screenshot tool with on-canvas annotation
> and a real CLI** — Qt-based GUI capture (rectangles, arrows,
> blur/pixelate redaction, freehand, text, numbered markers, color
> picker) wrapped in a daemon (`flameshot &`) plus subcommands
> (`flameshot gui`, `flameshot screen`, `flameshot full`,
> `flameshot launcher`, `flameshot config`) that script cleanly
> from a hotkey or a CI job — pinned to **v13.3.0** (released
> 2025-10-28, [LICENSE](https://github.com/flameshot-org/flameshot/blob/v13.3.0/LICENSE),
> SPDX `GPL-3.0-or-later`).

Source: <https://github.com/flameshot-org/flameshot>

## TL;DR

The Linux/macOS/Windows screenshot space is a layered mess:
GNOME `gnome-screenshot` and KDE `spectacle` cover the basics but
ship no annotation flow worth using; `scrot` / `maim` / `grim` +
`slurp` capture pixels and stop there; `Shutter` (Perl, GTK2,
unmaintained for years on modern distros); `ksnip` (good
annotation, weaker scripting); macOS `screencapture` is a
one-shot binary with no on-canvas editing; Windows Snipping Tool
is mouse-only. `flameshot` is the right shape: one daemon, one
hotkey, drag a rectangle, annotate inline, then **upload-to-
Imgur**, **copy-to-clipboard**, **save-to-file**, **pin-on-screen**,
or **pipe-to-stdout** in the same gesture — and every one of
those terminal verbs is also a subcommand you can wire into a
script (`flameshot gui --raw | tesseract - -` for instant OCR,
`flameshot full -p ~/screens/` for a cron-driven kiosk capture).

## Install

```bash
# macOS
brew install --cask flameshot

# Debian / Ubuntu
sudo apt install flameshot

# Arch
sudo pacman -S flameshot

# Fedora
sudo dnf install flameshot

# Pre-built binaries (.AppImage, .deb, .rpm, Windows installer,
# macOS .dmg) for v13.3.0 live at:
#   https://github.com/flameshot-org/flameshot/releases/tag/v13.3.0
```

Hard prereqs: a running compositor (X11 or `xdg-desktop-portal`-
equipped Wayland — Sway/Hyprland/KDE Plasma/GNOME 45+); on
Wayland, screenshot via the portal interface is required because
flameshot cannot read other clients' surfaces directly.

## Common invocations

```bash
# Interactive: hotkey-driven rectangle + annotate + copy
flameshot gui

# One-shot whole-screen, no GUI, save with timestamped name
flameshot full -p ~/screens/

# Capture only the screen the cursor is on, copy to clipboard
flameshot screen -c

# Delay 5s (for menus/tooltips), save + copy
flameshot gui -d 5000 -c -p ~/screens/

# Pipe raw PNG to a downstream tool (OCR, upload, attach)
flameshot gui --raw | tesseract stdin stdout

# Reconfigure the on-canvas tool palette (which tools, which
# default colors, hotkey bindings)
flameshot config

# Headless launcher for tray-less environments
flameshot launcher
```

## Why orthogonal to existing zoo

The zoo has terminal recorders (`asciinema`, `vhs`), media
optimizers (`oxipng`, `pngquant`, `jpegoptim`), and a region
selector (`grim`-adjacent shells exist) but **no on-canvas
annotated-screenshot tool** — every prior entry that touches
images either records video, optimizes a finished file, or
captures pixels without an editing surface. `flameshot` fills
the "drag-rectangle → annotate → copy" loop that bug reports,
PR review screenshots, and tutorial assets all need.

## Caveats

- Wayland support requires `xdg-desktop-portal` and the
  compositor's screencast portal to be functional; bare-metal
  `wlroots` without portals (older Sway configs) fall back to
  X11-only behavior or fail.
- Imgur upload is anonymous + unauthenticated by default; for
  long-lived links use a self-hosted upload endpoint via the
  config dialog (it speaks any HTTP POST endpoint).
- macOS Cask requires manually granting **Screen Recording**
  permission in System Settings → Privacy & Security on first
  launch; first-shot otherwise returns a black image.
- GPL-3.0-or-later: redistributions and forks must carry source;
  the upstream binary ships are GPL-clean but vendoring the Qt
  GUI bits into a proprietary product is the wrong move.
- The daemon (`flameshot &`) holds a Qt event loop ~25-50 MB
  resident; on a tiling WM with `xdg-portal` already running
  this is negligible, on an i3 + bar-only setup it is the
  largest GUI process you'll have.
