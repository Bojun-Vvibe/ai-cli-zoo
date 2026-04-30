# lapce

> **Lightning-fast modal-friendly code editor written in Rust** — a
> native GPU-accelerated GUI editor (built on the Druid / Floem
> toolkit, wgpu rendering) with built-in LSP, DAP debugging, vim
> keybindings, remote development over SSH, and a WASI plugin
> system. Pinned to **v0.4.6**
> ([LICENSE](https://github.com/lapce/lapce/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/lapce/lapce>

## TL;DR

Lapce is a from-scratch desktop code editor that targets the
"VSCode's features, native binary's startup time" sweet spot.
Render path is wgpu (Vulkan / Metal / DX12 under the hood), so
scrolling a 50k-line file stays at refresh-rate even on integrated
graphics; cold start is sub-second on a modern laptop. LSP and DAP
are first-class — point it at any `language-server` binary in
config and you get diagnostics, completions, go-to-def, rename,
inline hints, debug breakpoints. Remote development over SSH works
without a separate "server" install: Lapce ships a small proxy that
copies itself to the remote host on first connect, then renders the
remote workspace locally while editing happens server-side. The
plugin system runs WASI modules so extensions are sandboxed and
language-agnostic — Rust / Go / TypeScript / Zig all compile to
the same `.wasm` plugin format.

## Install

```bash
# Homebrew (macOS / Linux)
brew install --cask lapce

# Arch Linux
pacman -S lapce

# Or grab a release binary
curl -L https://github.com/lapce/lapce/releases/download/v0.4.6/lapce-macos.dmg -o lapce.dmg

# From source
cargo install --locked --git https://github.com/lapce/lapce lapce-app
```

## Example

```bash
# Open a project
lapce ~/code/my-rust-app

# Edit a remote workspace over SSH (requires SSH key auth set up)
# Use the "Connect to SSH Host" command palette entry, or:
lapce --new ssh://user@host/path/to/project
```

## When to use

- You want VSCode's keymap + LSP UX but in a single ~30 MB native
  binary that opens instantly and stays responsive on huge files.
- You edit code over SSH frequently and the VSCode Remote-SSH
  round-trip latency bothers you.
- You like vim keybindings as a first-class mode (not a plugin
  emulating them).
- You want a sandboxed plugin system — WASI plugins cannot read
  arbitrary files on your disk the way an Electron extension can.

## When NOT to use

- You depend on a specific VSCode extension — the WASI plugin
  ecosystem is much smaller and many editor features people expect
  (Jupyter notebooks, Live Share, GitHub Copilot the product, big
  vendor language packs) do not have Lapce equivalents yet.
- You need a stable, conservative tool — Lapce is pre-1.0 and
  breaking config / keymap changes do land between minor versions.
- You want a TUI editor — Lapce is a GUI app; reach for Helix or
  Neovim if you live in a terminal.
