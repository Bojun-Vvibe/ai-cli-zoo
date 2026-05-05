# greenclip

> **A simple, single-binary clipboard manager that pairs with
> `rofi` / `dmenu` / `wofi` to give Linux a real clipboard
> history without a Qt/GTK panel applet** — runs as a daemon
> (`greenclip daemon`), watches the X11 PRIMARY and CLIPBOARD
> selections, persists every entry to a flat history file, and
> exposes that history as one-entry-per-line on stdout
> (`greenclip print`) so any picker that reads stdin can become
> the UI. Pinned to **v4.2** (SPDX: `BSD-3-Clause`,
> [LICENSE](https://github.com/erebe/greenclip/blob/master/LICENSE)).

Source: <https://github.com/erebe/greenclip>

## TL;DR

`greenclip` is the right reach when "I want clipboard history,
on Linux, without committing to a desktop environment" is the
problem. The daemon is one Haskell-compiled static binary
(~6 MB), zero runtime dependencies, and it does exactly three
things: watch selections, persist them, print them. Everything
about the *picker* — fuzzy matching, preview, theming,
keybindings — is delegated to whatever you already use to pick
things (rofi, dmenu, wofi, fzf in a terminal, even
[`skim`](../skim/) in tmux), because clipboard history is a
selection problem and Linux already has good selection
primitives.

## Install

```bash
# Arch Linux (AUR)
yay -S greenclip

# Nix
nix-env -iA nixpkgs.haskellPackages.greenclip

# Static binary from releases
curl -L -o ~/.local/bin/greenclip \
  https://github.com/erebe/greenclip/releases/download/v4.2/greenclip
chmod +x ~/.local/bin/greenclip

# Verify
greenclip --help
```

## Usage

```bash
# 1. Run the daemon (foreground; usually launched by systemd --user
#    or your i3 / sway autostart)
greenclip daemon

# 2. Print history, one entry per line, newest first
greenclip print

# 3. Wire into rofi as a clipboard picker (the canonical recipe)
rofi -modi "clipboard:greenclip print" -show clipboard \
     -run-command '{cmd}'

# 4. Same idea with dmenu
greenclip print | dmenu -i -l 20 | xargs -r echo -n | xclip -sel clip

# 5. Same idea with fzf in a terminal
greenclip print | fzf --no-sort | tr -d '\n' | xclip -sel clip

# 6. Clear the history (useful before pasting a screenshare)
greenclip clear

# 7. Pin a specific entry so it never expires from history
#    (interactive — opens picker, marked entries get a star)
rofi -modi "clipboard:greenclip print" -show clipboard \
     -kb-custom-1 "Alt+p"   # rofi binding emits exit-code 10 → pin
```

## Why it matters

Clipboard history on Linux is a chronically half-solved problem.
The full desktop environments (KDE Klipper, GNOME's clipboard
indicator) ship one, but they bring 200 MB of dependencies and
only work inside that DE. The standalone alternatives split into
**heavy GUI managers** (CopyQ — Qt, scriptable, full window with
preview pane, ~80 MB resident) and **scripted nothings** (a
hand-rolled `xclip` + `tail` + `awk` pipeline that loses entries
on reboot). `greenclip` sits in the gap: one static binary, one
daemon, one history file, one print verb — and the picker is
whatever you already trust to pick things. The result is
clipboard history that costs ~6 MB of disk, ~10 MB of RAM, and
fits a tiling-WM aesthetic where every utility is a small
composable program.

## Niche It Fills

**Selection-primitive clipboard history for Linux tiling WMs.**
The space splits three ways: GUI clipboard managers (Klipper,
GNOME indicator, CopyQ — fine, but DE-coupled or heavyweight),
home-rolled `xclip` scripts (fast to write, lose history on
reboot, no deduplication), and `greenclip` (daemon + flat file +
stdin-friendly print verb). It is the canonical pick in i3 /
sway / bspwm / herbstluftwm setups precisely because it lets the
existing rofi / dmenu / wofi launcher *be* the clipboard UI.

## Vs Already Cataloged

- **Vs [`clipse`](../clipse/):** clipse is the modern Bubble Tea
  TUI clipboard history (Go, full-screen TUI with image preview).
  greenclip is the X11-daemon-plus-rofi-picker recipe (no TUI of
  its own, picker is whatever you already use). Pick clipse for
  a self-contained TUI app launched from a hotkey; pick greenclip
  for "rofi is already my launcher and I want it to have a
  clipboard mode too".
- **Vs [`wl-clipboard`](../wl-clipboard/):** wl-clipboard is
  primitives (`wl-copy` / `wl-paste`) for Wayland — set and read
  the current clipboard, no history. greenclip is the *history*
  layer above primitives (X11-side; for Wayland, pair `cliphist`
  + `wl-clipboard` + rofi for the equivalent stack).
- **Vs CopyQ (not cataloged):** CopyQ is the heavyweight pick
  with a Qt window, scripting (`copyq eval`), per-entry tagging,
  and image previews. greenclip trades all of that for "one
  binary, one daemon, picker is rofi". Pick CopyQ for power-user
  scripting and image preview; pick greenclip for tiling-WM
  minimalism.

## Caveats

- **X11-only.** greenclip uses X11 selection-watching APIs.
  Wayland users want `cliphist` (Go, also a daemon + print
  recipe) which is the morally equivalent tool on the Wayland
  side. greenclip will not run on `WAYLAND_DISPLAY`-only
  sessions, including most modern GNOME and KDE defaults.
- **Slow release cadence.** v4.2 is from 2021; the project is
  feature-complete and bug-fix-only. The static binary still
  works on current X11 stacks because the X11 selection API is
  stable, but expect no new features.
- **Plaintext history file.** History sits in `~/.cache/greenclip.history`
  as raw plaintext (or binary serialized — depending on
  version). Anything you copy — passwords, tokens, 2FA codes —
  is written to disk. Pair with a "clipboard purge" hotkey
  (`greenclip clear`) before screenshares, and consider
  excluding password-manager copies (most password managers
  expose a "clear after N seconds" feature that defeats *any*
  clipboard manager — that is the point).
- **No image / file-list support.** greenclip stores text
  entries only (UTF-8 selection content). Screenshots copied as
  image data fall through to the live clipboard but are not
  added to history. Use clipse / CopyQ when image history
  matters.
- **Rofi / dmenu picker is your responsibility.** If your
  launcher is broken or unstyled, greenclip looks broken too —
  the daemon is fine, but the UX is the picker's. The flip side
  is that any rofi theme you already use applies for free.
