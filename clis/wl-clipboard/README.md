# wl-clipboard

> **The Wayland-native answer to `xclip` / `xsel` — two tiny
> C binaries (`wl-copy`, `wl-paste`) that talk the Wayland
> clipboard / primary-selection protocols directly, so
> `echo hi | wl-copy` and `wl-paste | grep …` work the same
> way they did under X11**, with first-class MIME-type
> handling (`--type image/png`), watch mode (`wl-paste -w
> command`) for clipboard-managers, and a primary-selection
> flag (`-p`) that maps to the middle-click buffer. Pinned
> to **v2.3.0**
> ([COPYING](https://github.com/bugaevc/wl-clipboard/blob/v2.3.0/COPYING),
> GPL-3.0-or-later).

Source: <https://github.com/bugaevc/wl-clipboard>

## TL;DR

On a Wayland session there is no `DISPLAY` for `xclip` to
talk to, and the `wl_data_device` protocol requires a
short-lived foreground process to *own* the selection until
someone pastes — so the X11-era `echo … | xclip -sel clip`
muscle memory breaks in unobvious ways (your clipboard goes
empty when the shell exits). `wl-clipboard` is the
two-binary answer: `wl-copy` reads stdin, registers as the
selection owner, and forks a background helper that
keeps the data alive until the next `wl-copy` overwrites
it; `wl-paste` reads the current selection back out. MIME
handling is real (`wl-copy --type image/png < screenshot.png`,
`wl-paste --list-types`, `wl-paste --type text/html`), the
primary selection (middle-click) is a separate buffer
addressed with `-p`, and `wl-paste --watch <cmd>` invokes
`<cmd>` every time the selection changes — the building
block every Wayland clipboard-manager (`clipman`,
[`clipse`](../clipse/), `cliphist`) is built on. Works on
every mainline Wayland compositor (`sway`, `Hyprland`,
`river`, GNOME, KDE Plasma, `wayfire`, `niri`); under XWayland
it falls back through the standard data-device bridge so
mixed X11/Wayland clients see the same buffer.

## Install

```bash
# Debian / Ubuntu
sudo apt install wl-clipboard

# Arch
sudo pacman -S wl-clipboard

# Fedora
sudo dnf install wl-clipboard

# Homebrew (Linux only — macOS has no Wayland)
brew install wl-clipboard

# Build from source (Meson, libwayland-client)
git clone --depth 1 --branch v2.3.0 \
  https://github.com/bugaevc/wl-clipboard.git
cd wl-clipboard && meson setup build && meson compile -C build
```

## Usage

```bash
# Copy / paste (clipboard selection, ctrl-c / ctrl-v buffer)
echo "hello wayland" | wl-copy
wl-paste

# Primary selection (highlight-then-middle-click buffer)
echo "primary text" | wl-copy -p
wl-paste -p

# Copy a PNG, paste it back into a file
wl-copy --type image/png < screenshot.png
wl-paste --type image/png > restored.png

# Inspect what's currently on the clipboard
wl-paste --list-types

# Watch the clipboard and run a command on every change
# (this is how clipboard-history daemons are built)
wl-paste --watch tee -a ~/.cache/clipboard.log

# Clear the clipboard
wl-copy --clear

# Cross-platform shell function for portable scripts
copy() {
  if   command -v wl-copy   >/dev/null; then wl-copy
  elif command -v xclip     >/dev/null; then xclip -selection clipboard
  elif command -v pbcopy    >/dev/null; then pbcopy
  fi
}
echo something | copy
```

## Why it's interesting

X11's clipboard ICCCM contract was awkward but stable for
30 years — `xclip` / `xsel` papered over it well enough that
shell pipes into the clipboard were a solved problem. The
Wayland equivalent is *protocol-correct* but adds a hard
constraint: the process that called `set_selection` must stay
alive to actually *serve* the buffer when a paste request
arrives. That single change broke every shell snippet that
piped into a clipboard tool and exited; `wl-clipboard`'s
fork-and-detach helper is the small infrastructure piece
that makes Wayland's clipboard usable from a shell at all.
Beyond that it's the canonical primitive everything else
in the Wayland clipboard ecosystem is built on:
[`clipse`](../clipse/) / `cliphist` / `clipman` all subscribe
via `wl-paste --watch`; screenshot tools (`grim` / `slurp`)
pipe images into `wl-copy --type image/png`; password
managers (`pass`, `bitwarden-cli`) use it for the
30-second-then-clear pattern (`echo $secret | wl-copy &&
sleep 30 && wl-copy --clear`). Pick over a wrapper like
`xsel-wayland-shim` when you want the real protocol surface
(MIME types, primary selection, watch mode); pair with
[`clipse`](../clipse/) or `cliphist` for clipboard history,
with `grim` / `wf-recorder` for screenshot pipelines, with
`pass` / `bitwarden-cli` for ephemeral-credential paste.
Caveats — Wayland-only (no Windows, no macOS, no plain X11
session — keep an `xclip`/`pbcopy` fallback in shell
functions), GPL-3.0-or-later (fine for normal use, awkward
for closed-source bundling), the watch-mode helper relies on
the compositor honouring the data-control protocol (most do;
GNOME historically lagged on the `wlr-data-control`
extension — modern versions are fine, but very old GNOME
sessions need the `xdg-desktop-portal` fallback path).
