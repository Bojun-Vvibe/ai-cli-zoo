# grim

> **Grab images from a Wayland compositor — the screenshot primitive that `scrot` / `maim` are to X11, written for the `wlr-screencopy` protocol so it works on every wlroots-based compositor (`sway`, `hyprland`, `river`, `wayfire`, `cosmic-comp`, `labwc`) without per-compositor shims.**
> A small C utility by Simon Ser (`emersion`) — co-author of the wlroots
> compositor library and the Wayland protocol extensions it implements —
> that captures the framebuffer of one or all outputs, optionally cropped
> to a geometry, and writes a PNG / JPEG / PPM to stdout or a file.
> Pinned to **v1.4.0**
> ([LICENSE](https://github.com/emersion/grim/blob/master/LICENSE),
> MIT).

Source: <https://github.com/emersion/grim>

## TL;DR

On X11, the universal screenshot primitive is `scrot` / `maim` /
`import` (ImageMagick) — pipe to `xclip`, save to a file, hand to an
OCR tool, done. On Wayland, `scrot` / `maim` no longer work because
the X server is gone (or running as XWayland and only sees X11
clients), so a freshly migrated user discovers their entire
"screenshot a region, OCR it, paste the text into the LLM prompt"
workflow has silently broken.

`grim` is the Wayland-native replacement. It speaks the
`wlr-screencopy` protocol — the same protocol every wlroots
compositor (`sway`, `hyprland`, `river`, `wayfire`, `labwc`,
`cosmic-comp`) advertises — so one binary covers the entire
wlroots-flavored Wayland ecosystem. Output is PNG by default, JPEG
or PPM via `-t`, and writes to stdout when the path is `-`, which is
the property that makes it composable with the rest of the Unix
toolkit:

```bash
grim - | wl-copy -t image/png       # screenshot → clipboard
grim -g "$(slurp)" - | tesseract - - # region → OCR → stdout
grim ~/screenshots/$(date +%F-%T).png
```

The companion tool [`slurp`](https://github.com/emersion/slurp) (also
emersion) provides the interactive region-selection that grim
intentionally omits, and the two are designed to be piped together
— grim handles capture, slurp handles selection, and the
`grim -g "$(slurp)"` idiom is the canonical "screenshot a region"
recipe across the wlroots ecosystem.

## Install

```bash
# Arch Linux (extra)
sudo pacman -S grim slurp wl-clipboard

# Debian / Ubuntu
sudo apt install grim slurp wl-clipboard

# Fedora
sudo dnf install grim slurp wl-clipboard

# From source (meson + ninja, ~30 s build)
git clone https://github.com/emersion/grim
cd grim && meson setup build && ninja -C build
sudo ninja -C build install

# verify
grim -h
# Usage: grim [options...] [output-file]
```

Runtime requirements: a wlroots-based Wayland compositor that
exposes `wlr-screencopy-unstable-v1` (sway, hyprland, river,
wayfire, labwc, cosmic-comp all do). Does **not** work on
GNOME-Shell or KDE Plasma's Wayland sessions out of the box —
those compositors implement different screenshot protocols
(`xdg-desktop-portal-gnome`, `xdg-desktop-portal-kde`) and need
portal-aware tools instead (`flameshot`, `gnome-screenshot`,
`spectacle`).

## Usage

```bash
# 1) Screenshot all outputs to a timestamped PNG
grim ~/Pictures/$(date +%Y%m%d-%H%M%S).png

# 2) One specific output (e.g. only the laptop's internal display)
grim -o eDP-1 ~/Pictures/laptop.png

# 3) Region-select with slurp, copy to clipboard
grim -g "$(slurp)" - | wl-copy -t image/png

# 4) Region-select, OCR, pipe the text into an LLM CLI for explanation
grim -g "$(slurp)" - \
  | tesseract - - \
  | llm "What is this error message telling me, and how do I fix it?"

# 5) Capture, write to stdout, pipe through a compressor for size
grim - | oxipng - -o 4 > screenshot.png
```

## Niche & tradeoffs

`grim` lives in the narrow but load-bearing slot of "Wayland
screenshot primitive for wlroots compositors." Distinct from:

- **X11 incumbents** (`scrot`, `maim`, `import`, `gnome-screenshot`
  on X11) — those depend on the X protocol and are silently
  unusable under a Wayland session, even when the user thinks they
  still work because XWayland is running. grim is the migration
  target.
- **GNOME / KDE-portal screenshot tools** (`gnome-screenshot`,
  `spectacle`, `flameshot`) — those go through
  `xdg-desktop-portal-gnome` / `xdg-desktop-portal-kde` and work
  *only* on GNOME / KDE Wayland sessions, where grim does not work.
  Pick by compositor: grim for wlroots, portal-tools for
  GNOME / KDE.
- **Region-selection tools** ([`slurp`](https://github.com/emersion/slurp))
  — slurp picks a geometry, grim captures it; the two compose as
  `grim -g "$(slurp)" file.png` and neither replaces the other.
- **Screen *recorders*** (`wf-recorder`, `wl-screenrec`) —
  capture a video stream rather than a single frame; grim is
  still-image only, and the recorders complement rather than
  replace it.
- **Clipboard utilities** ([`wl-clipboard`](https://github.com/bugaevc/wl-clipboard)
  for `wl-copy` / `wl-paste`) — grim writes the bytes, wl-copy
  puts them on the clipboard; the canonical
  "screenshot to clipboard" one-liner needs both.

The right mental model is "**`scrot` for the Wayland era** — one
~50KB C binary, no GUI, no config file, no daemon, output goes
where stdout goes." Keep it on every Wayland-using box for the
30-line shell-script productivity layer (region OCR, LLM-prompt
attachments, bug-report attachments, dotfiles previews) where a
desktop-environment screenshot tool is overkill and a portal
dependency is fragile.

Caveats — (1) Wayland-only, and specifically wlroots-only;
GNOME / KDE Wayland sessions need different tools, and pure X11
sessions need `scrot` / `maim`. (2) No interactive UI of any
kind — no rectangle selection, no annotation, no upload. Pair
with `slurp` for selection and an annotator like
[`satty`](https://github.com/gabm/Satty) or `swappy` if the
workflow needs arrows / blur boxes / text overlays.
(3) `wlr-screencopy-unstable-v1` is, as the name implies,
"unstable" — the protocol has been stable in practice for years
across wlroots, but the version namespace remains and a future
protocol bump may require a grim version bump in lockstep with
your compositor.
