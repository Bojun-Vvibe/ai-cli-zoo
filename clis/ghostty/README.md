# ghostty

> **A fast, native, GPU-accelerated terminal emulator.**
> A Zig-built terminal from Mitchell Hashimoto that targets
> "feels native on every platform, runs every TUI correctly,
> stays out of your way" — full xterm compatibility, libghostty
> as an embeddable core, and zero JavaScript anywhere in the
> render path. Pinned to **v1.3.1**
> ([LICENSE](https://github.com/ghostty-org/ghostty/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ghostty-org/ghostty>

## TL;DR

`ghostty` is the terminal emulator you reach for when you want
*the* most-conformant xterm, native windowing on macOS / Linux,
and GPU-accelerated rendering — without paying the Electron
tax. The architecture is split into two:

1. **`libghostty`** — a Zig library implementing the terminal
   state machine, parser, font shaping, and renderer. Designed
   to be embedded in *other* programs (Neovim's experimental
   embedded terminal, IDE panels, custom multiplexers).
2. **`ghostty`** — the standalone application that wraps
   `libghostty` with a native UI: AppKit on macOS, GTK 4 on
   Linux. There is no Windows binary yet (planned via WinUI).

Configuration is a single `~/.config/ghostty/config` file with
~100 documented keys (`font-family`, `theme`, `keybind = …`,
`background-opacity`, `quick-terminal-position` for a Yakuake-
style drop-down). Reload on save, no restart.

## Install

```bash
# macOS (official .dmg)
# https://release.files.ghostty.org/1.3.1/Ghostty.dmg
brew install --cask ghostty   # also tracks the official cask

# Arch Linux
pacman -S ghostty

# Fedora (COPR)
dnf copr enable pgdev/ghostty && dnf install ghostty

# Nix
nix profile install nixpkgs#ghostty

# Source build (Zig 0.13+ required)
git clone --depth 1 -b v1.3.1 https://github.com/ghostty-org/ghostty
cd ghostty && zig build -Doptimize=ReleaseFast
./zig-out/bin/ghostty

# verify
ghostty +version
```

A minimal `~/.config/ghostty/config` to get started:

```
font-family = JetBrainsMono Nerd Font
font-size   = 14
theme       = catppuccin-mocha
background-opacity = 0.95
keybind     = cmd+t=new_tab
keybind     = cmd+d=new_split:right
```

## License

MIT — see
[LICENSE](https://github.com/ghostty-org/ghostty/blob/main/LICENSE).
Permissive; the `libghostty` core is intentionally MIT-licensed
so other projects (editors, IDEs, multiplexers) can embed it
without copyleft obligations. Bundled fonts and icons carry
their own licenses, listed under `pkg/`.

## One Concrete Example

```bash
# 1. List what compile-time options the binary was built with
ghostty +show-config --default --docs | head -40
# Dumps every config key with its default and doc string —
# the canonical source of truth for "which knob does what".

# 2. Print the config the running build is actually using
ghostty +show-config

# 3. Open a one-off ghostty pointed at a specific config file
ghostty --config-file=/tmp/demo.config

# 4. SSH into a host where the remote tput doesn't know "ghostty"
TERM=xterm-256color ssh older-box
# Or, better, copy the terminfo over once:
infocmp -x ghostty | ssh older-box 'tic -x -'
# Now $TERM=ghostty works on the remote and full features
# (true-colour, bracketed paste, OSC 52 clipboard) light up.

# 5. Quick terminal mode (Yakuake-style drop-down)
# In ~/.config/ghostty/config:
#   keybind = global:cmd+grave_accent=toggle_quick_terminal
# Press Cmd-` from anywhere to drop a terminal sheet from the top
# of the screen; press again to hide. Useful for "I just need to
# paste one git command" without context-switching apps.

# 6. Image rendering via the Kitty graphics protocol
# ghostty implements the Kitty graphics protocol, so:
kitten icat photo.png    # renders inline
# (works because ghostty advertises the same protocol)
```

## Niche It Fills

**The "actually-native, actually-fast, actually-correct"
terminal gap.** The modern terminal emulator landscape is split
between three uncomfortable corners: Electron-based ones
(slow, large RAM footprint), web-tech wrappers (mediocre
rendering, broken edge cases), and 1990s X11 emulators
(perfectly fast, hostile UX, no ligatures or true colour).
`ghostty` aims for the empty fourth corner: native windowing
toolkits, GPU rendering, full xterm + Kitty + iTerm2 protocol
compatibility, and zero web stack. For developers running TUIs
all day (`tmux`, `neovim`, `lazygit`, `k9s`, `htop`, the dozens
of TUIs in this catalog), the result is "every TUI just works
correctly, including the obscure escape sequences", with input
latency that competes with `alacritty` and feature breadth that
competes with `kitty`.

For an LLM-CLI workflow, `ghostty` is the **host terminal** —
when an agent opens a TUI (a picker, a diff viewer, a long
streaming response), it's the layer that decides whether
ligatures, true-colour syntax highlighting, and bracketed
paste actually work. Its strict xterm conformance means
agent-driven TUIs render the same on every machine where
`ghostty` runs.

## Why use it

Three things `ghostty` does that other terminals don't:

1. **Strict, documented xterm conformance.** The terminal state
   machine is a separately-tested library with a giant corpus
   of escape-sequence test cases. TUIs that misrender on
   `kitty` (for protocol-violation reasons) tend to render
   correctly here, and bug reports get a parser fix rather
   than a "feature added".
2. **Single config file, hot-reloaded.** Edit, save, the next
   surface (tab, split, window) picks up the change. No GUI
   preferences pane to drift out of sync, no per-profile
   spaghetti.
3. **`libghostty` as embeddable core.** The terminal isn't
   married to the app. Neovim's experimental embedded
   terminal, custom IDE panels, and future multiplexers can
   reuse the same parser/renderer Zig library, so the
   ecosystem doesn't fork once per host.

## Vs Already Cataloged

- **Vs [`kitty`](../kitty/):** Closest peer. `kitty` is older
  (2017), Python-extensible, with its own protocol extensions
  (graphics, keyboard, remote control) that `ghostty` partly
  adopts. `ghostty` is younger (2024 public beta), strictly
  native (no Python), and prioritises xterm conformance over
  novel protocols. Pick `kitty` for its plugin ecosystem and
  cross-platform coverage including Windows; pick `ghostty`
  for raw native feel on macOS/GTK and conformance.
- **Vs [`wezterm`](../wezterm/):** `wezterm` is Rust + Lua,
  cross-platform including Windows, scriptable for almost any
  customisation. `ghostty` is leaner — no scripting layer, no
  Lua, configuration is a flat file. Pick `wezterm` when you
  want to script the terminal itself; pick `ghostty` when the
  scripting surface is a *liability* (less to break, less to
  maintain).
- **Vs [`tabby`](../tabby/):** `tabby` is Electron-based with
  built-in SSH client, settings UI, plugin marketplace.
  `ghostty` is the opposite design: no SSH client (use `ssh`),
  no settings UI (edit the file), no plugins (embed
  `libghostty` if you need to extend). Pick `tabby` for the
  app-store experience; pick `ghostty` for minimal,
  fast, native.

## Caveats

- **No Windows binary yet.** v1.3.1 ships macOS and Linux
  (GTK). Windows support is on the roadmap (planned via WinUI)
  but not shipping. WSL users run the Linux build inside WSLg
  with mixed results.
- **GTK 4 on Linux means modern distros only.** Building or
  packaging on GTK 3-only systems will fail; Ubuntu 22.04+,
  Fedora 38+, Arch current are the safe baselines.
- **Zig toolchain churn for source builds.** Pin to the Zig
  version listed in the release's `flake.nix` or `build.zig`;
  newer/older Zig tends to break the build until upstream
  catches up.
- **Config keys evolve.** The file format is stable but new
  keys land per release and a few legacy aliases are
  gradually deprecated. Run `ghostty +show-config --default
  --docs` after upgrading to see the current canonical set.
- **No tabs/splits in the headless `libghostty`.** Tabs and
  splits are the *application* layer's responsibility (AppKit
  on macOS, GTK Notebook on Linux). Embedders get the
  terminal cell grid; UI chrome is on them.
- **It is a terminal, not a multiplexer.** For session
  persistence across SSH disconnects, panes that survive a
  laptop reboot, or attach-from-multiple-clients, you still
  want `tmux` / `zellij` running *inside* `ghostty`.
