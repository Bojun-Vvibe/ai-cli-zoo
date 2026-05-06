# pacaptr

> Snapshot date: 2026-05. Upstream: <https://github.com/rami3l/pacaptr>

**One Pacman-style command surface in front of every package
manager you actually have to deal with.**
pacaptr is a single Rust binary that translates an Arch-style
`-S`/`-R`/`-Syu`/`-Ss`/`-Qi` verb grammar into the native invocation
of whichever package manager owns the box (Homebrew, MacPorts,
APT, DNF, Zypper, Pacman, XBPS, APK, Pkg, Chocolatey, Scoop, WinGet,
pip, conda, npm, cargo, gem, Flatpak, Snap, and a long tail more).
You memorise *one* set of flags and stop context-switching between
`brew install`, `apt-get install`, `dnf install`, `winget install`,
`choco install`, `pkg install`, `pip install --upgrade`, etc.

## Repo + version + license

- Repo: <https://github.com/rami3l/pacaptr>
- Latest release: **`v0.23.1`** (2025-11-09)
- License: **GPL-3.0** —
  <https://github.com/rami3l/pacaptr/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Rust

## Install

```bash
# Homebrew (macOS / Linux)
brew install rami3l/tap/pacaptr

# Cargo from source
cargo install pacaptr

# Or pull a prebuilt binary from the GitHub Releases page (Linux,
# macOS x86_64/aarch64, Windows x86_64).

# Daily verbs — same on every OS
pacaptr -S ripgrep              # install
pacaptr -R ripgrep              # uninstall
pacaptr -Syu                    # refresh sources + upgrade everything
pacaptr -Ss markdown            # search
pacaptr -Qi ripgrep             # show installed package info
pacaptr -Sw ripgrep             # download only, don't install
pacaptr --using brew -S fd      # force a specific backend
pacaptr --dryrun -Syu           # print the underlying command, don't run
pacaptr --using pip -S httpx    # treat pip as "the system package manager"
```

## Niche

The "**Pacman verb grammar as a portable shim over every other
package manager**" slot. Where [`topgrade`](../topgrade/) is a
*meta-upgrader* that runs every package manager's update step in
sequence, pacaptr is the inverse: a *single command grammar* you
script against, that picks the right backend at runtime. Useful for:

- **Cross-platform dotfiles / install scripts** — one
  `pacaptr -S git neovim ripgrep fd bat` line works on a fresh Mac,
  a fresh Ubuntu box, a fresh Arch box, a fresh Windows box with
  Scoop, and a fresh FreeBSD box, with no per-OS branching.
- **Polyglot per-language manager normalisation** — `--using pip`,
  `--using cargo`, `--using npm`, `--using gem`, `--using conda`
  let you treat language ecosystems as first-class package managers
  inside the same script, so a "set up this dev box" recipe can
  speak one verb set across system + language layers.
- **Muscle-memory portability** — if you came from Arch and now
  work on macOS / Ubuntu / Windows, you keep typing
  `-S` / `-Syu` / `-Qi` / `-Rns` and pacaptr translates.
- **Forensic / dry-run mode** — `--dryrun` prints the exact
  underlying command (`brew install ripgrep`, `apt-get install -y
  ripgrep`, etc.) without running it, so you can paste it into a
  README, a Dockerfile, or a Justfile with confidence.

## Why it matters

- **30+ backends, autodetected** — Homebrew, MacPorts, APT, DNF,
  Zypper, Pacman, XBPS (Void), APK (Alpine), Pkg (FreeBSD),
  Chocolatey, Scoop, WinGet, pip, pipx, conda, mamba, npm, pnpm,
  yarn, cargo, gem, Flatpak, Snap, Tlmgr, Emerge (Gentoo), Slackpkg,
  and more. pacaptr inspects the host and picks the default; you
  override with `--using <backend>` per invocation or
  `PACAPTR_DEFAULT_BACKEND=...` per shell.
- **Faithful flag mapping, not lossy aliases** — every supported
  verb (`-S`, `-Sw`, `-Sg`, `-Sl`, `-Sii`, `-Q`, `-Qc`, `-Qe`,
  `-Qm`, `-Qo`, `-Qs`, `-R`, `-Rn`, `-Rns`, `-U`, `-F`, etc.) maps
  to the *closest semantic equivalent* in each backend, with the
  README documenting each translation table so behaviour is
  predictable rather than magical.
- **Single Rust binary** — no Python, no Ruby, no daemon; the
  binary is small, has no runtime dependencies beyond the package
  manager it shells out to, and ships prebuilt for Linux / macOS /
  Windows on every release.
- **`--dryrun` and `--no-confirm` for CI** — the same command
  shape works under `set -e` automation: `--dryrun` for diff-style
  review, `--yes` / `--no-confirm` for unattended runs.
- **Active in 2025** — `v0.23.1` (2025-11-09) is the most recent
  release at snapshot time; the project has shipped roughly
  quarterly for several years and accepts new backends via small
  trait implementations under `src/pm/`.
- **Honest scope** — pacaptr is *not* a package manager. It does
  not maintain its own repository, does not resolve dependencies,
  and does not cache. It is a thin, well-tested translation layer.
  When the underlying manager is broken, pacaptr inherits the
  breakage; when it is fast, pacaptr is fast. That clarity is the
  reason it stays at one binary and one config file.
