# clipse

> **A configurable TUI clipboard manager** — single Go binary
> that runs as a background daemon, persists clipboard history
> (text + images) to a local JSON store, and pops a Bubble Tea
> picker on demand. Pinned to **v1.1.1**
> ([LICENSE](https://github.com/savedra1/clipse/blob/main/LICENSE),
> MIT).

Source: <https://github.com/savedra1/clipse>

## TL;DR

`clipse` is the keyboard-driven clipboard manager that finally
exists outside the GNOME / KDE applet world. A `clipse
--listen-shell` daemon watches the system clipboard
(`pbpaste`/`wl-paste`/`xclip`), appends every change to a JSON
history file, and a separate `clipse` invocation opens a TUI
picker (Bubble Tea, fuzzy filter, vim keys) that lets you
search, preview, pin, and re-copy any past entry — including
images, which are previewed inline in terminals that support
the Kitty graphics protocol. Pinned items survive eviction
when the history hits its size cap; everything else rolls
off in FIFO order. Theming is a single `~/.config/clipse/custom_theme.json`
and key bindings are remappable.

## Install

```bash
# Homebrew (macOS / Linux)
brew install clipse

# Go install
go install github.com/savedra1/clipse@v1.1.1

# Arch (AUR)
yay -S clipse

# Verify
clipse --version    # clipse v1.1.1
```

## Example usage

```bash
# 1. First run creates ~/.config/clipse/{config.json,clipboard_history.json}
clipse --help

# 2. Start the listener as a backgrounded daemon (Wayland / X11 / macOS)
#    macOS: relies on pbpaste; Wayland: needs wl-clipboard;
#    X11: needs xclip or xsel.
clipse --listen-shell &

# 3. Open the picker (bind this to a hotkey in your WM / hotkey daemon)
clipse
#    Inside the TUI:
#      / or ctrl+f   fuzzy filter
#      enter         copy selected back to clipboard and quit
#      p             pin / unpin current entry (survives eviction)
#      x             delete current entry from history
#      ?             help

# 4. systemd user unit so the listener auto-starts
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/clipse.service <<'UNIT'
[Unit]
Description=clipse clipboard listener
After=graphical-session.target

[Service]
ExecStart=%h/.local/bin/clipse --listen-shell
Restart=on-failure

[Install]
WantedBy=default.target
UNIT
systemctl --user daemon-reload
systemctl --user enable --now clipse.service

# 5. Cap history at 200 entries, keep images for 24h
#    Edit ~/.config/clipse/config.json:
#      { "maxHistory": 200, "imageDisplay": { "type": "kitty" },
#        "historyFile": "clipboard_history.json" }

# 6. Hotkey examples
#    sway:   bindsym $mod+v exec foot -e clipse
#    hyprland: bind = SUPER, V, exec, kitty -e clipse
#    macOS skhd: cmd - v : open -a Alacritty --args -e clipse
```

## Why this lives in the zoo

The terminal ecosystem is full of clipboard *bridges*
(`pbcopy`, `xclip`, `wl-copy`, `osc52` shims) but very few
clipboard *managers* that don't drag in a desktop environment.
`clipse` is the right size: a tiny Go daemon plus a Bubble Tea
picker, no DBus, no tray, no Electron, image support optional.
It works the same on macOS, Wayland, and X11 because it
delegates to whichever native bridge is installed, which makes
it the rare "clipboard history" tool you can actually put in
a cross-platform dotfiles repo.
