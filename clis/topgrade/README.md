# topgrade

> **Upgrade everything on your system with one command.** Pinned to
> **v17.4.0**, GPL-3.0
> ([LICENSE](https://github.com/topgrade-rs/topgrade/blob/main/LICENSE)).

- **Repo:** https://github.com/topgrade-rs/topgrade
- **Latest version:** v17.4.0
- **License:** GPL-3.0 (`LICENSE` at repo root, SPDX `GPL-3.0`)
- **Category:** `system-admin` / `dev-experience`
- **Language:** Rust

## What it does

`topgrade` is a meta-upgrade tool that detects every package
manager, language toolchain, plugin manager, and shell framework
installed on your machine and runs each one's update command in
sequence — Homebrew, MacPorts, apt/dnf/pacman/zypper, Flatpak,
Snap, `rustup`, `cargo install-update`, `npm -g`, `pnpm -g`,
`pipx`, `uv tool upgrade`, `gem`, `go install` re-runs, `nvim`
plugin managers (lazy.nvim, packer, vim-plug), tmux's `tpm`,
`oh-my-zsh`, `gh extension upgrade`, VSCode/VSCodium extensions,
container images via `docker`/`podman`, kernel + firmware on
Linux, macOS Software Update, and ~80 more — controlled by a
TOML config (`~/.config/topgrade.toml`) that lets you skip,
reorder, or insert custom pre/post hook steps. Reports a colored
summary at the end with per-step success/failure so you can see
what needs manual attention.

## Why included

The "Sunday morning chore" automation. Owning a Mac with brew +
rustup + nvm + pipx + a tmux config + a neovim plugin manager
means at least a dozen `update` commands a week; `topgrade` runs
all of them in dependency-correct order from one shell call and
exits non-zero if anything failed, which makes it equally useful
in a personal cron job. Community-maintained fork of the original
`topgrade` (now archived) actively shipping releases.
