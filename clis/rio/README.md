# rio

> **Rio** — a hardware-accelerated GPU terminal emulator written in
> Rust, targeting macOS / Linux / Windows / BSD desktops *and* a
> WebAssembly build that runs the same renderer in a browser tab.
> Pinned to **v0.4.2**, MIT — license file:
> [LICENSE](https://github.com/raphamorim/rio/blob/main/LICENSE).

Source: <https://github.com/raphamorim/rio>

## TL;DR

`rio` is a *terminal emulator*, not a TUI app — it is the window
that hosts your shell, peer to Alacritty / WezTerm / Kitty / Ghostty
/ Foot / iTerm2 / the macOS default Terminal.app. The renderer is
written on top of `wgpu` so the same draw path runs against Metal on
macOS, Vulkan on Linux, DirectX 12 on Windows, and WebGPU in the
WebAssembly target — one pipeline, four backends, no per-platform
fallback rendering code.

What earns the slot in a catalog already containing
[`ghostty`](../ghostty/), [`wezterm`](../wezterm/),
[`kitty`](../kitty/), and [`alloy`](../alloy/):

- **WebGPU + WASM target** — the same renderer ships as a browser
  build (you can embed a real terminal into a web page without
  xterm.js's canvas / DOM tradeoffs).
- **Per-tab split layouts and a navigation system** (`Bookmark`,
  `BottomTab`, `TopTab`, `Plain`) configurable from
  `~/.config/rio/config.toml` — closer to WezTerm's pane model than
  Alacritty's "one terminal per window".
- **Sugarloaf** — its custom text-rendering crate that batches glyph
  draws on the GPU; the practical effect is sub-frame redraws on
  scrollback flush even at 4K with a programming font.
- **Adaptive theme** that follows the OS dark / light setting
  without a restart.

## Install

```bash
# Homebrew cask (macOS)
brew install --cask rio

# Cargo from source
cargo install --git https://github.com/raphamorim/rio --locked

# Pre-built binaries from releases
# https://github.com/raphamorim/rio/releases/tag/v0.4.2
```

## Example commands / config

```bash
# Launch
rio

# Launch with a specific working directory and command
rio --working-dir ~/code/myproj -e "tmux new-session -A -s dev"
```

Minimal `~/.config/rio/config.toml`:

```toml
theme = "tokyo-night"
[fonts]
family = "JetBrains Mono"
size = 14
[window]
opacity = 0.95
blur = true
[navigation]
mode = "BottomTab"
```

## Niche it occupies

**GPU-accelerated, cross-backend, WASM-targetable terminal
emulator** — pick `rio` over [`alacritty`](../alacritty/) when you
want native splits and tabs without `tmux` / `zellij` underneath,
over [`wezterm`](../wezterm/) when you prefer a TOML config over a
Lua scripting surface, over [`kitty`](../kitty/) when the WebGPU /
browser embedding story matters, and over [`ghostty`](../ghostty/)
when you specifically want a Rust + `wgpu` codebase to hack on
rather than Zig + platform-native APIs. Orthogonal to multiplexers
([`tmux`](../tmuxinator/), [`zellij`](../zellij/),
[`abduco`](../abduco/)) — those run *inside* whichever emulator you
pick.

## Citation

- Repo: <https://github.com/raphamorim/rio>
- Latest release: **v0.4.2**
- License: **MIT**
- License file: [LICENSE](https://github.com/raphamorim/rio/blob/main/LICENSE)
