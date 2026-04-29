# macchina

> **A fast, minimalist system-information fetcher in Rust** — the
> spiritual successor to `neofetch` / `screenfetch` rebuilt around
> "start in <100 ms, render a themable two-column block, expose
> every line as a configurable widget." Pinned to **v6.4.0**
> (homebrew `macchina`),
> [LICENSE](https://github.com/Macchina-CLI/macchina/blob/main/LICENSE),
> MIT.

Source: <https://github.com/Macchina-CLI/macchina>

## TL;DR

`macchina` answers "what's this box, and what's it doing right now"
in roughly the time it takes a shell to print a prompt. Run it
bare and you get a logo on the left and a stack of fields on the
right: host name, OS, kernel, distro, uptime, memory, swap, CPU
model + load, GPU(s), shell, terminal, packages installed across
detected package managers, battery, local IP, public IP (opt-in).
What separates it from the older `*fetch` family is everything
around that core readout: themes are TOML files (`themes/`) that
control box-drawing characters, separator strings, key padding,
ASCII vs image logo, and per-field colors; the field list is a
configurable `Readouts = [...]` array so you can drop or reorder
lines per host; backends are written in Rust against per-OS
syscalls instead of shelling out, so a cold run on macOS / Linux
/ FreeBSD / NetBSD / Windows / Android (Termux) finishes in tens
of milliseconds rather than the multi-hundred-millisecond `bash +
fork-heavy` runs of `neofetch`. `macchina --doctor` explains why
any field is missing (permission, unsupported platform, optional
dep absent), which makes it usable as a portable smoke test for
"can this VM see its battery / GPU / package manager at all."

## Install

```bash
# Homebrew (macOS / Linux)
brew install macchina

# Cargo
cargo install macchina --locked

# Linux package managers
# Arch: pacman -S macchina  (or AUR: macchina-bin)
# Nix: nix-env -iA nixpkgs.macchina
# Void: xbps-install macchina

# Windows
# scoop install macchina

# from a release binary
curl -Lo macchina "https://github.com/Macchina-CLI/macchina/releases/download/v6.4.0/macchina-linux-x86_64"
chmod +x macchina && sudo install macchina /usr/local/bin/

# verify
macchina --version    # macchina 6.4.0
```

## What it does

- Two-column "logo + readouts" terminal banner, themable via TOML.
- Per-OS native backends (Linux, macOS, BSDs, Windows, Android/Termux).
- Configurable field list — drop, reorder, rename any line.
- `--doctor` mode prints why a field is empty (perms, platform,
  missing optional dep) instead of silently skipping it.
- Custom readouts: shell out to a command and embed its first line
  as a labeled row.
- ASCII or image (kitty / sixel) logos; bring-your-own ASCII art.
- JSON / plain output for piping into status bars or dashboards.

## pew-related use cases

- **Per-host startup banner** in `~/.zshrc` / `~/.bashrc` so every
  new shell prints "where am I, what kernel, how much free RAM"
  in <100 ms — cheap enough to keep on, unlike `neofetch`.
- **Inventory script across a fleet of dev VMs**: `ssh host
  macchina --output json` gives you a structured row per host
  (CPU, RAM, kernel, distro, package count) without parsing
  `uname -a` + `/proc/meminfo` by hand.
- **Smoke test for a fresh VM image**: `macchina --doctor` flags
  missing GPU / battery / package-manager visibility before the
  user notices.
- **Status-bar feed** (tmux, polybar, waybar): pipe the JSON
  output through `jq` to surface uptime / memory / load with the
  same formatting on every OS.
- **Recording / screenshot prep** (asciinema, freeze): single
  command that produces a clean, themed identity card for the
  top of a demo without the visual noise of `neofetch`.
