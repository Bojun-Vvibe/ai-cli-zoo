# fastfetch

> **Maintained C rewrite of `neofetch`: prints a system-info card
> (OS, kernel, uptime, packages, shell, GPU, memory, theme, …) next
> to a distro logo, in tens of milliseconds, with a JSON-driven
> module/preset system instead of a 10 kLoC bash file.** Pinned to
> **2.62.1**, MIT
> ([LICENSE](https://github.com/fastfetch-cli/fastfetch/blob/dev/LICENSE)).

- **Repo:** https://github.com/fastfetch-cli/fastfetch
- **Latest version:** 2.62.1
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `system-info` / `terminal-aesthetics`
- **Language:** C
- **Install:** `brew install fastfetch`,
  `apt install fastfetch` (Debian 13+), or prebuilt release tarball

## What it does

`fastfetch` reads platform-specific data sources directly (sysctl
/ `/proc` / `/sys` / IORegistry / WMI) instead of forking dozens
of helper binaries the way `neofetch` did, so a full render is
typically 10–40 ms versus 200–800 ms. Output is composed from
**modules** — `OS`, `Kernel`, `Uptime`, `Packages`, `Shell`,
`Display`, `WM`, `Theme`, `Icons`, `Font`, `Cursor`, `Terminal`,
`CPU`, `GPU`, `Memory`, `Swap`, `Disk`, `LocalIp`, `Battery`,
`PowerAdapter`, `Locale`, `Weather`, `PublicIp`, `Player`, … plus
custom `Command` modules — declared in a JSONC config
(`~/.config/fastfetch/config.jsonc`) with per-module key colour,
format string, and ordering. Preset configs (`fastfetch -c
neofetch`, `-c paleofetch`, `-c all`) reproduce the look of older
tools or dump every available field for debugging. Logos are
selected by distro auto-detect or `--logo <name>`, and accept ANSI,
sixel, kitty, or iterm image protocols for true-colour output in
modern terminals.

## Why included

`neofetch` was archived in 2024 and never gained Wayland-correct
display detection, current macOS metadata, or fast startup —
making it unsuitable as a "terminal greeter" or as a structured
inventory tool in scripts. `fastfetch` is the active replacement
the major distros (Arch, Fedora, Debian 13, Homebrew) standardised
on: it runs fast enough to put in a shell prompt or `tmux`
status-right, and `fastfetch --format json` emits a stable
machine-readable inventory of host facts that an automation layer
can diff across machines. For an LLM-CLI workflow asking "what
host am I on, what GPU, how much RAM", `fastfetch --format json`
is one call instead of five fragile `uname` / `system_profiler` /
`lspci` invocations.
