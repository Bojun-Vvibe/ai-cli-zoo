# wezterm

> **A GPU-accelerated cross-platform terminal emulator + multiplexer in
> one binary, configured by a Lua file (`~/.config/wezterm/wezterm.lua`)
> instead of a TOML schema — same binary on macOS / Linux / Windows /
> FreeBSD, native ligatures + colour-emoji + IME, image protocols
> (iTerm2 + Sixel + Kitty), built-in multiplexer (panes + tabs +
> workspaces) so `tmux` / `zellij` is optional, SSH-as-domain
> (`wezterm ssh user@host`) and serial-as-domain so a remote session
> reuses local Lua bindings, and a programmable key / mouse / event
> system in Lua (any keybind can be a Lua callback that inspects the
> current pane's title / cwd / process and decides what to do).** Pinned
> to **20240203-110809-5046fc22**
> ([LICENSE.md](https://github.com/wez/wezterm/blob/main/LICENSE.md),
> MIT).

Source: <https://github.com/wez/wezterm>

## TL;DR

`wezterm` is what to reach for when the daily-driver terminal needs
to be the *same* terminal on macOS, Linux, and Windows — same
config file, same keybinds, same colour scheme, same font shaping
— and the existing options each have a gap (iTerm2 is macOS-only,
Alacritty has no multiplexer + no Lua + GUI config is per-platform,
Windows Terminal is Windows-only, Kitty is great but config is a
custom DSL not a real scripting language). The whole config lives
in one Lua file you check into dotfiles, the multiplexer is built
in (so `wezterm` + `wezterm cli split-pane --right` is enough
without `tmux` / `zellij` if you want one tool not two), `wezterm
ssh user@host` reuses *all* of your local keybinds and Lua bindings
on the remote pane (including OSC 7 cwd tracking and OSC 133
prompt marking for `Ctrl+Up/Down` jump-to-prev-prompt), and the
GPU renderer (WebGPU / Metal / Vulkan / D3D11) is fast enough for
high-refresh-rate scrollback even with full ligature shaping
through Harfbuzz. The downside is install footprint (the binary is
~30 MB; macOS package is ~80 MB with bundled icons + assets) and
that the version number is a date-stamp not a semver, so "latest
stable" is the date string `20240203-110809-5046fc22`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install --cask wezterm           # macOS GUI bundle
brew install wezterm                  # Linux CLI

# Linux package managers
apt install wezterm                   # Debian / Ubuntu (>= 24.04)
dnf install wezterm                   # Fedora
pacman -S wezterm                     # Arch
nix-env -iA nixpkgs.wezterm           # Nix

# Windows
winget install wez.wezterm
scoop install wezterm

# Direct binary (any platform)
# https://github.com/wez/wezterm/releases/tag/20240203-110809-5046fc22

# verify
wezterm --version                     # 20240203-110809-5046fc22 ...
```

## License

**MIT** — see
[LICENSE.md](https://github.com/wez/wezterm/blob/main/LICENSE.md).
Permissive; vendoring or shipping the binary inside another product
is fine. Bundled fonts (JetBrains Mono, Roboto, etc.) carry their
own permissive licences; a list is in
[`docs/licenses.md`](https://github.com/wez/wezterm/blob/main/docs/licenses.md).

## One Concrete Example

```lua
-- ~/.config/wezterm/wezterm.lua — full minimal config
local wezterm = require 'wezterm'
local act = wezterm.action
local config = wezterm.config_builder()

config.font = wezterm.font_with_fallback({ 'JetBrains Mono', 'Symbols Nerd Font' })
config.font_size = 13.0
config.color_scheme = 'Tokyo Night'
config.window_decorations = 'RESIZE'
config.use_fancy_tab_bar = false
config.scrollback_lines = 50000
config.audible_bell = 'Disabled'

-- multiplexer keybinds — pane splits + workspace switcher
config.keys = {
  { key = '|',  mods = 'CMD|SHIFT', action = act.SplitHorizontal { domain = 'CurrentPaneDomain' } },
  { key = '-',  mods = 'CMD',       action = act.SplitVertical   { domain = 'CurrentPaneDomain' } },
  { key = 'h',  mods = 'CMD',       action = act.ActivatePaneDirection 'Left'  },
  { key = 'l',  mods = 'CMD',       action = act.ActivatePaneDirection 'Right' },
  { key = 'k',  mods = 'CMD',       action = act.ActivatePaneDirection 'Up'    },
  { key = 'j',  mods = 'CMD',       action = act.ActivatePaneDirection 'Down'  },
  -- programmable: open a Lua-driven workspace switcher
  { key = 's',  mods = 'CTRL|SHIFT', action = act.ShowLauncherArgs { flags = 'WORKSPACES' } },
}

-- programmable event: rewrite tab title from the active pane's process
wezterm.on('format-tab-title', function(tab)
  local p = tab.active_pane.foreground_process_name or ''
  return { { Text = ' ' .. (p:match('([^/]+)$') or 'shell') .. ' ' } }
end)

-- SSH multiplexing as a first-class domain
config.ssh_domains = {
  { name = 'staging-bastion', remote_address = 'bastion.staging.example' },
}

return config
```

```bash
# CLI surface — drive the running GUI from a script / keybind
wezterm cli list                         # panes / tabs / windows JSON
wezterm cli split-pane --right -- htop   # split + run command
wezterm cli send-text --pane-id 5 'ls\n'

# launch into a named workspace
wezterm start --workspace=incident-2026-04 --cwd ~/runbooks

# SSH as a domain — local Lua keybinds work in the remote pane
wezterm ssh user@host

# serial console (embedded / boards) — same Lua bindings
wezterm serial /dev/cu.usbserial-1410 --baud 115200
```

## Niche It Fills

**The "one terminal binary, same Lua config, every OS — with a
built-in multiplexer so `tmux` is optional, and an SSH domain so
the remote session reuses local keybinds" slot.** The gap is real:
iTerm2 is macOS-only and configured by GUI panels (not dotfiles);
Alacritty is fast and cross-platform but has no multiplexer and
its YAML config has no programmable event hooks; Kitty has a
multiplexer (`kittens`) but its DSL is custom (not Lua) and macOS
support is second-class; Windows Terminal is Windows-only.
`wezterm` is the only mainstream choice that ships *one binary*
running natively on macOS / Linux / Windows / FreeBSD with *one
Lua config* in dotfiles, *one built-in multiplexer*, and *first-
class SSH-as-a-domain* so `wezterm ssh host` reuses your full
local keybinding set on the remote pane.

## Why use it

Three things `wezterm` does that no other terminal in this catalog
does together in one binary:

1. **Lua-as-config, not a custom DSL.** Every keybind, mouse
   binding, hyperlink rule, tab title formatter, status-bar
   widget, and event handler is a Lua callback. `wezterm.on(
   'format-tab-title', function(tab) ... end)` lets the tab title
   reflect the active process; `wezterm.on('user-var-changed',
   function(window, pane, name, value) ... end)` reacts to OSC 1337
   user variables emitted by your shell prompt; `wezterm.on(
   'open-uri', function(window, pane, uri) ... end)` lets you
   intercept URL clicks (e.g. open `jira:ABC-123` in the right
   browser). No other terminal in this catalog has this depth of
   programmability without a plugin system on top.
2. **SSH-as-a-domain reuses local keybinds remotely.** `wezterm
   ssh user@host` is not just `ssh user@host` in a pane — the
   remote pane is a *first-class wezterm pane* whose keybinds,
   colour scheme, copy / paste, and OSC 7 / OSC 133 handling all
   work as if it were local. Combined with `wezterm.mux` (the
   multiplexer is a separate process you can detach from and
   reattach to), you can have a long-running remote session that
   survives local laptop sleeps and reattaches to the same panes
   on wake — without `tmux` / `screen` on the remote side.
3. **Image + ligature + emoji + IME, all native.** Sixel + iTerm2
   + Kitty image protocols are all supported (so `chafa` /
   `viu` / `imgcat` all work); ligatures shape correctly through
   Harfbuzz (so `JetBrains Mono` / `Fira Code` / `Cascadia Code`
   render `=>`, `!=`, `>>=` as expected); colour emoji and CJK
   IME work out of the box on every OS. Other cross-platform
   terminals (Alacritty, st, foot) drop one or more of these.

For a daily driver that has to be the same on a macOS laptop, a
Linux desktop, and a Windows VM, with one config in dotfiles, one
keybind set, and one multiplexer story, `wezterm` is the only
binary that covers all three OSes natively without compromise.

## Vs Already Cataloged

- **Vs [`zellij`](../zellij/):** different layer, sometimes
  overlapping. `zellij` is a *terminal multiplexer* (panes + tabs
  + sessions inside any terminal emulator); `wezterm` is a
  *terminal emulator* with multiplexing built in. They compose: if
  you already use `zellij` for its layouts and plugin system, run
  it inside `wezterm` and disable wezterm's pane bindings. If you
  want one binary instead of two, wezterm's built-in multiplexer
  covers the common cases (splits, tabs, named workspaces,
  detachable mux sessions over SSH).
- **Vs [`helix`](../helix/) / [`nushell`](../nushell/):** these
  run *inside* wezterm. Wezterm is the substrate; helix is the
  editor; nushell is the shell. A common modern stack is
  `wezterm` → `nushell` → `helix`, all configured in dotfiles,
  all cross-platform. Wezterm's `wezterm.on('user-var-changed')`
  hook composes with shell prompts that emit OSC 1337 user vars
  (nushell can do this) for a reactive status bar.
- **Vs `Alacritty` / `Kitty` / `iTerm2` / Windows Terminal (not
  cataloged):** wezterm is the cross-platform answer with the
  most programmable config. Alacritty wins on lowest install
  footprint and config-file simplicity (YAML, no scripting); Kitty
  wins on macOS-only ligature rendering quality and the
  `kittens` plugin ecosystem; iTerm2 wins on macOS-native polish;
  Windows Terminal wins on Windows integration. Wezterm wins on
  "I refuse to learn three different terminals."

## Caveats

- **Version number is a date stamp, not semver.** The latest
  stable tag is `20240203-110809-5046fc22` (date + time + short
  SHA). Newer commits ship as nightly builds with their own date-
  SHA tags. Pin to a specific date-tag in CI; `latest` will move.
- **Install footprint is large.** The macOS app bundle is ~80 MB
  (bundled fonts, icons, assets); the Linux binary is ~30 MB.
  This is normal for a GPU terminal with a built-in multiplexer
  and embedded Lua runtime, but if you want a 2 MB single-binary
  terminal, look at `foot` (Linux/Wayland-only) or stay on
  Alacritty.
- **Lua config means a Lua bug breaks startup.** `wezterm.lua`
  errors fail to a fallback config and print a notification, but
  a typo in a frequently-edited file is a real friction. Develop
  config changes in a side window with `wezterm --config-file
  /tmp/test.lua` before promoting.
- **Built-in multiplexer is good, not great vs `tmux` / `zellij`.**
  Detach / reattach works, but the plugin ecosystem (status-bar
  widgets, layouts, session managers) is far smaller than tmux's
  TPM ecosystem or zellij's plugin system. If you live in
  multiplexer plugins, run zellij / tmux *inside* wezterm.
- **GPU acceleration occasionally trips driver bugs.** On older
  Intel iGPUs / certain Linux Wayland compositors, the WebGPU /
  Vulkan path can render glitches; fall back to `front_end =
  'Software'` in the Lua config to use the CPU renderer (slower
  but bulletproof). Documented in the upstream FAQ.
- **OSC 7 / OSC 133 + cwd tracking + jump-to-prompt require shell
  cooperation.** The features are wezterm-side; emitting the OSC
  sequences is shell-side. zsh / bash / fish need a small prompt
  snippet from `wezterm shell-completion --shell zsh` (or the
  upstream integration docs) to enable cwd tracking, prompt
  jumping, and "open new pane in current dir." Without it,
  wezterm degrades gracefully but the killer keybinds don't
  light up.
