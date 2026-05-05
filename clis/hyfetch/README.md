# hyfetch

> **A `neofetch` fork that prints the system info banner with a
> Pride-flag colour gradient over the ASCII distro logo, plus
> an optional Python-3 rewrite (`hyfetch -m fast`) that fixes
> a pile of upstream bugs and ships ~30 new distro logos.**
> One Python package, two binaries: `hyfetch` (the gradient
> wrapper) and `neowofetch` (a maintained successor to the
> archived `neofetch` Bash script). Pinned to **2.0.5**
> ([LICENSE.md](https://github.com/hykilpikonna/hyfetch/blob/master/LICENSE.md),
> MIT).

Source: <https://github.com/hykilpikonna/hyfetch>

## TL;DR

`neofetch` (dylanaraps) was the canonical "screenshot my desktop
with the distro logo + system info" tool for years. The original
project was archived in 2024 and stopped accepting fixes. `hyfetch`
is the practical answer: a Python wrapper that calls the Bash
neofetch (or its bundled rewrite `neowofetch`) and re-colours the
distro-logo ASCII art with one of ~30 Pride-flag gradients
(`rainbow`, `transgender`, `nonbinary`, `bisexual`, `pansexual`,
`asexual`, `lesbian`, `genderfluid`, `agender`, `aromantic`,
`gay-men`, `intersex`, …). Run `hyfetch -c` once for an
interactive picker that writes `~/.config/hyfetch.json` (flag
choice, distro override, colour mode) and from then on `hyfetch`
prints the banner with the chosen gradient over the auto-detected
distro art. `hyfetch -m fast` switches to the Python rewrite
(`neowofetch` engine) which is several times faster than the Bash
script on first run, supports more distros (Asahi, Bazzite,
NixOS-on-Darwin, Vanilla OS, etc.), and fixes long-standing
neofetch bugs around macOS `system_profiler` parsing and
WSL detection.

## Install

```bash
# pipx (recommended — isolated venv, single command on $PATH)
pipx install hyfetch

# pip (user install)
pip install --user hyfetch

# Homebrew (macOS / Linux)
brew install hyfetch

# Arch (community)
pacman -S hyfetch

# Debian / Ubuntu (in main repos as of 24.04)
sudo apt install hyfetch

# from source (Python 3.8+)
pip install git+https://github.com/hykilpikonna/hyfetch.git@2.0.5
```

After install, run:

```bash
hyfetch -c          # one-time interactive config (flag, mode, colours)
hyfetch             # print system info with the chosen gradient
neowofetch          # the bundled neofetch successor with no gradient
```

## What it actually is

Two things in one package:

1. **`hyfetch` — gradient wrapper.** A ~3 kLOC Python program
   that runs `neofetch` (or the bundled `neowofetch`) under the
   hood, captures the distro-logo ASCII art before it hits the
   terminal, and rewrites every ANSI colour escape with one of
   the bundled Pride-flag gradients. Output is otherwise
   byte-identical to upstream neofetch — same fields, same
   layout, same right-side info column — so any screenshot-bot,
   blog-post template, or `~/.config/i3/i3blocks.conf` that
   already parses neofetch keeps working.
2. **`neowofetch` — the maintenance fork.** A drop-in
   replacement for the archived `neofetch` Bash script, kept on
   life support and steadily ported toward Python. Adds ~30 new
   distro logos (Asahi, Bazzite, Nobara, Vanilla OS, Cachy,
   Garuda, Endeavour rebrand, Pop_OS! variants, NixOS-on-Darwin,
   …), fixes macOS detection on Apple Silicon (correct chip name,
   correct uptime under `launchd`), and restores WSL2 distro
   detection that the archived upstream got wrong on recent
   Windows builds.

## When to choose

- **You want the screenshot, not a dependency.** `hyfetch`
  has no daemon, no service, no background process — it is a
  one-shot CLI that prints a banner and exits. Drop it in
  `~/.zshrc` / `~/.bashrc` / `fish_greeting` if you want the
  banner on every new shell, or call it manually before a
  screenshot.
- **You miss `neofetch` and the upstream is archived.** The
  maintained alternative ecosystem fragmented (`fastfetch`,
  `macchina`, `pfetch`, `nitch`, `ufetch`, dozens of single-
  distro toys); `neowofetch` is the conservative answer that
  keeps the original neofetch CLI shape and config-file
  format, so existing `~/.config/neofetch/config.conf` files
  keep working with one symlink (`ln -s neowofetch
  /usr/local/bin/neofetch`).
- **You want the Pride-flag gradient.** This is the singular
  feature `hyfetch` adds on top of every other system-fetch
  tool. None of the active alternatives ship this out of the
  box; rolling it yourself means writing a 24-bit ANSI
  re-colourer over the distro-logo grid.

## Vs already cataloged

- **Vs [`fastfetch`](../fastfetch/) (active, C, MIT):** `fastfetch`
  is the performance-oriented neofetch successor — JSON-config-
  driven, faster cold start, larger logo library, no Bash, no
  Python. Pick `fastfetch` when speed and a static binary matter
  most. Pick `hyfetch` when you want the Pride-flag colour
  gradient on top of the logo and don't mind the Python
  runtime; the two coexist fine on the same machine.
- **Vs [`macchina`](../macchina/) (active, Rust, MPL-2.0):**
  `macchina` is a Rust rewrite with a fundamentally different
  output style — minimalist, small bar charts, theme files
  rather than a logo-plus-info layout. Pick `macchina` when you
  prefer the minimalist look; pick `hyfetch` when you want the
  classic neofetch look with gradient logos.
- **Vs [`cpufetch`](../cpufetch/):** orthogonal — `cpufetch`
  prints CPU-only info with a vendor-logo banner; `hyfetch` is
  whole-system. Both can sit side-by-side in the shell startup
  block.

## Caveats

- **Python runtime cost.** `hyfetch` startup is ~150–250 ms on
  a modern laptop (Python interpreter + import + neofetch
  shell-out). If you put it in `fish_greeting` and open many
  shells per minute, prefer `hyfetch -m fast` (uses
  `neowofetch`'s Python engine, no Bash sub-shell) or switch
  to `fastfetch` for sub-50 ms cold start.
- **Gradient depends on truecolor terminal.** 24-bit colour is
  required for the gradient to render correctly. Inside `tmux`
  set `set -ga terminal-overrides ",*256col*:Tc"`; inside
  `screen` truecolor is awkward — use `tmux` instead.
- **`neofetch` upstream is archived.** `hyfetch` ships
  `neowofetch` as the maintained successor, but the surrounding
  community-distro logo files in `/usr/share/neofetch/` may go
  stale; pin to the `hyfetch` package version (which bundles
  its own logo set) rather than relying on the distro
  `neofetch` package.
- **Not a monitor.** `hyfetch` prints once and exits — for
  live system stats use [`btop`](../btop/) /
  [`bottom`](../bottom/) / [`glances`](../glances/) /
  [`gtop`](../gtop/); for repeated polling in a status bar
  use [`fastfetch --pipe`](../fastfetch/) or
  [`macchina`](../macchina/).
- **Config file location varies.** `~/.config/hyfetch.json`
  on Linux/macOS, `%APPDATA%\hyfetch.json` on Windows; the
  underlying `neofetch` config is still at
  `~/.config/neofetch/config.conf` and `hyfetch` honours it
  for fields, layout, and ASCII art selection (the gradient
  is the only thing `hyfetch` overrides).
