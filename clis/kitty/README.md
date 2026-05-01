# kitty

- **Repo:** https://github.com/kovidgoyal/kitty
- **Version:** 0.40.1 (current stable on the 0.40.x line)
- **License:** GPL-3.0 — see [`LICENSE`](https://github.com/kovidgoyal/kitty/blob/master/LICENSE)
- **Language:** C + Python (OpenGL-accelerated rendering core in C; configuration, kittens, and remote-control plumbing in Python; uses platform OpenGL/EGL on Linux/X11/Wayland and CoreGraphics on macOS)
- **Install:** `brew install --cask kitty` (macOS) · `curl -L https://sw.kovidgoyal.net/kitty/installer.sh | sh /dev/stdin` (official cross-distro installer to `~/.local/kitty.app`) · `apt install kitty` / `dnf install kitty` / `pacman -S kitty` · ships one binary plus a `kitten` companion launcher for the bundled tool suite (`kitten icat`, `kitten ssh`, `kitten diff`, `kitten hyperlinked-grep`, etc.)

## Overview

`kitty` is a GPU-accelerated terminal emulator that
treats the terminal as a programmable surface, not a
glass TTY. The renderer compiles glyph atlases on the
GPU and pages them through a tile shader, so a 4K
window scrolling a `cat` of a 200 MB log file stays at
the monitor's refresh rate while a CPU-only emulator
turns into a slideshow. Beyond raw speed, it ships
**three protocol extensions** that have started to
become de-facto standards across modern emulators:
the **kitty graphics protocol** (in-band PNG / RGBA
image transfer with z-index, sub-rect placement, and
animation; consumed by `kitten icat`, `nvim` image
plugins, `chafa --format=kitty`, `matplotlib`,
`euporie`, `mpv --vo=kitty`), the **kitty keyboard
protocol** (disambiguated key events including
Shift-Tab, Ctrl-i vs Tab, modifier-only press / release,
international layouts), and **OSC 8 hyperlinks** with
clickable, addressable spans rendered correctly. On
top of that, kitty has **native tabs and split layouts
with no multiplexer** (Stack / Tall / Fat / Grid /
Splits / Horizontal / Vertical), a **remote-control
socket** (`kitty @ launch --type=tab nvim`,
`kitty @ set-colors`, `kitty @ scroll-window`) so
scripts can drive the terminal from the inside, and a
"kittens" extension model (Python plugins that run as
their own pane) that ships a curated set of high-value
tools out of the box: `kitten ssh` (transparently
copies your shell config + neovim + zsh into the
remote `$XDG_RUNTIME_DIR` so you get the same prompt
on the other side), `kitten icat` (display images
inline), `kitten diff` (side-by-side diff with image
support), `kitten hyperlinked-grep` (rg with clickable
file:line: links), `kitten clipboard` (programmatic
read/write of the system clipboard via OSC 52),
`kitten panel` (turn a kitty window into a desktop
panel widget), `kitten themes` (curated color-scheme
gallery with live preview).

## Niche

**A fast, GPU-accelerated terminal that treats the
terminal as a programmable surface — native tabs and
splits without tmux, in-band image protocol that
everything modern is starting to support, full
keyboard disambiguation, OSC 8 hyperlinks, a remote-
control socket scripts can drive from inside, and a
kittens plugin model that ships transparent-config
SSH, an image viewer, a diff tool, and a hyperlinked
ripgrep wrapper out of the box.** The role is "the
terminal you use when you actually want images and
plots inline in TUIs, when you want clickable
file:line: links from `rg` straight into `$EDITOR`,
when you want disambiguated key events for serious
modal editors, and when you want to script the
terminal from inside a shell session." Competing
universe: alacritty / wezterm / foot / ghostty /
iTerm2 / xterm. See comparisons below.

## When to use

- You want **inline images / plots** in TUIs and
  notebooks: `matplotlib` with the kitty backend,
  `euporie` notebooks with image cells, `chafa
  --format=kitty pic.png`, `kitten icat plot.png`,
  `mpv --vo=kitty video.mp4` all render in-band.
- You want **clickable file:line: links** from
  `rg` / `cargo` / `gcc` output straight into your
  editor: `kitten hyperlinked-grep PATTERN` emits
  OSC 8 hyperlinks; ctrl-click opens at the line.
- You want **disambiguated key events** for modal
  editors (Helix, Neovim with `vim.keycode`, Kakoune):
  the kitty keyboard protocol distinguishes
  Ctrl-i from Tab, Shift-Tab from Tab, and reports
  modifier press / release events that classical
  terminals collapse.
- You want **transparent SSH** that mirrors your
  local shell, prompt, neovim config, and aliases
  to the remote box without a per-host install:
  `kitten ssh user@host` copies your config into a
  scratch directory under `$XDG_RUNTIME_DIR` so the
  remote shell looks identical to the local one,
  and removes it on disconnect.
- You want **native tabs and split layouts without
  tmux** for local work: `Ctrl+Shift+T` for a tab,
  `Ctrl+Shift+Enter` for a window split, layouts
  cycle with `Ctrl+Shift+L`. (You still want tmux
  for *persistence* across SSH disconnects; kitty
  replaces tmux for *local* tiling.)
- You want **a remote-control socket** so scripts
  can drive the terminal: `kitty @ launch --type=tab
  --title=logs tail -F /var/log/syslog`, `kitty @
  set-colors --all ~/.config/kitty/themes/dark.conf`,
  `kitty @ scroll-window --match title:logs end`.
- You want **OSC 8 hyperlinks** rendered as real
  clickable spans rather than raw escape sequences.

## When NOT to use

- You want **the smallest, lowest-overhead, no-config
  GPU terminal** with zero scripting surface — pick
  [`alacritty`](../alacritty/) (if added). Alacritty
  is intentionally minimal: no tabs, no splits, no
  scripting, no graphics protocol; pair it with tmux
  for tiling. Pick alacritty when a 5 MB binary and a
  one-screen YAML file are the requirement.
- You want **a Lua-scriptable terminal with the
  richest config DSL** and built-in workspaces /
  domains for SSH multiplexing — pick
  [`wezterm`](../wezterm/). Wezterm's Lua config and
  domain abstraction beat kitty's Python `kitty.conf`
  for complex multi-host workflows; kitty is simpler.
- You're on Wayland-only and want **the most
  Wayland-native, smallest emulator** — pick `foot`
  (not in zoo). Foot is Wayland-native, sub-MB, and
  fast, but no graphics protocol and no scripting
  surface.
- You want **the closest thing to native macOS UX**
  with deep AppKit / Tab Bar / Quick Look integration
  — iTerm2 still wins on macOS-only polish, though
  kitty has caught up substantially on macOS in
  recent releases.
- You strictly need **a non-GPL license** for
  redistribution as part of a closed product —
  kitty is GPL-3.0; pick alacritty (Apache-2.0 / MIT)
  or wezterm (MIT) instead.

## Comparison vs alternatives in zoo

- [`wezterm`](../wezterm/) — Rust GPU terminal with
  a Lua config DSL, multiplexer-style "domains" for
  SSH, built-in mux that survives disconnects. Pick
  wezterm when you want the richest scripting + a
  domain abstraction for many SSH targets; pick kitty
  when you want the kitty graphics protocol (more
  widely supported in TUIs at this snapshot) and the
  kittens plugin model.
- [`asciinema`](../asciinema/) — terminal session
  recorder. Complementary — record any kitty session
  with `asciinema rec`, replay with `asciinema play`
  (or convert to GIF with [`agg`](../agg/) /
  [`vhs`](../vhs/)).
- [`vhs`](../vhs/) — programmatic terminal-recording
  that types into a headless terminal and emits a
  GIF / MP4. Complementary — vhs uses its own
  embedded terminal renderer; kitty is the
  interactive surface for humans.
- [`ttyd`](../ttyd/) — expose a local terminal over
  the web. Orthogonal — ttyd serves a browser-side
  xterm.js; kitty is the desktop surface.
- [`tmux`](../tmux/) (if added) — terminal multiplexer
  for persistence across SSH disconnects and remote
  pair-programming. Complementary — kitty handles
  *local* tabs / splits / layouts; tmux handles
  *persistence + remote attach*.
- [`fzf`](../fzf/) / [`skim`](../skim/) /
  [`television`](../television/) — fuzzy pickers.
  Complementary — kitten ssh + a kitten-aware fzf
  binding gives you "ctrl-r history search" that
  honours OSC 8 hyperlinks for selected commands.
- [`glow`](../glow/) / [`bat`](../bat/) — markdown
  / file viewers. They light up best in a terminal
  with images; kitty's graphics protocol lets glow
  embed actual rendered images from markdown,
  not ASCII placeholders.

## Why it earns a slot in an AI-native workflow

LLM-CLI workflows generate *artifacts* the agent
wants the human to look at quickly: a generated
matplotlib plot, a diff between two prompt-rewriting
strategies, a screenshot of a flaky e2e test, a
chart of token / latency / cost over the last 1000
runs, a rendered markdown report with embedded
images. With a classic terminal those artifacts
require an out-of-band viewer (`open plot.png`,
spawn a GUI, lose context). Kitty renders all of
that **in-band** — `matplotlib.use('module://kitty')`
puts the plot inline in the same scrollback as the
agent transcript, `kitten icat artifacts/run-1234/screenshot.png`
shows the screenshot under the agent message that
produced it, `kitten diff before/ after/` gives a
side-by-side image-aware diff. For agents that emit
hyperlinked references to files (`see src/foo.rs:42`),
the kitty keyboard protocol + OSC 8 means
`kitten hyperlinked-grep` and modern Helix / Neovim
configs make those references one-click navigable.
And `kitten ssh` means an agent that asks you to
"run this on the staging box" lands you in a remote
shell that has your local prompt, neovim config,
and aliases pre-staged, so the cognitive overhead of
the SSH hop disappears.

## Example invocations

```bash
# Inline image display
kitten icat artifacts/plot.png

# matplotlib inline (in any Python script run from kitty)
import matplotlib
matplotlib.use('module://matplotlib-backend-kitty')
import matplotlib.pyplot as plt
plt.plot([1,2,3]); plt.show()   # renders in the terminal

# Hyperlinked grep — clickable file:line: results
kitten hyperlinked-grep TODO src/

# Side-by-side image-aware diff between two trees
kitten diff before/ after/

# Transparent SSH that mirrors local shell config
kitten ssh user@host   # remote shell now uses your local zshrc + nvim

# Remote control: open a new tab running tail -F
kitty @ launch --type=tab --title=logs tail -F /var/log/syslog

# Live theme switch without restart
kitten themes Catppuccin-Mocha
kitty @ set-colors --all ~/.config/kitty/themes/dracula.conf

# Native tab + split workflow (no tmux)
#   Ctrl+Shift+T   new tab
#   Ctrl+Shift+Enter   new window split
#   Ctrl+Shift+L   cycle layout (Splits / Stack / Tall / Grid)

# Read / write the system clipboard from a script
printf "hello" | kitten clipboard
kitten clipboard --get-clipboard

# Turn a kitty window into a desktop panel (Wayland / X11)
kitten panel --edge=top --lines=2 yes "status line"
```
