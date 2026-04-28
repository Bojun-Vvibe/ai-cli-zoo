# fnm

- **Repo:** https://github.com/Schniz/fnm
- **Version:** v1.39.0 (released 2026-03-06)
- **License:** GPL-3.0 ([LICENSE](https://github.com/Schniz/fnm/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install fnm` · `cargo install fnm` · `winget install Schniz.fnm` · `curl -fsSL https://fnm.vercel.app/install | bash` · prebuilt static binaries on the GitHub release page · binary name is `fnm`

## Overview

`fnm` (Fast Node Manager) is a single-binary Node.js version
manager written in Rust. It does the same job as `nvm`
(install many Node versions side by side, switch between them
per shell, pin a project to a specific version via `.nvmrc` /
`.node-version`) but without `nvm`'s ~600 ms shell-startup
penalty: where `nvm` is a multi-thousand-line bash script that
sources on every interactive shell launch, `fnm` is a static
binary plus a small `eval "$(fnm env --use-on-cd)"` shim. The
practical result is a `~/.zshrc` that re-loads in tens of
milliseconds instead of half a second, and a `cd` into a
project root that swaps Node versions in well under 100 ms.

The flag list is intentionally small. `fnm install 22.21.1`
fetches an official Node tarball, verifies its checksum, and
unpacks it under `~/.local/share/fnm/node-versions/`. `fnm use
22.21.1` symlinks that version into the current shell's
ephemeral `PATH` (per-shell, not global, so two terminal tabs
can run different Node versions concurrently). `fnm default
22` writes a default symlink consulted on shell launch. `fnm
list` / `fnm list-remote` enumerate installed and downloadable
versions. `--use-on-cd` is the killer mode: drop a
`.node-version` or `.nvmrc` file at any level of the project
tree, and `cd` into that subtree silently activates the
declared version.

## Use cases

- **Polyglot or multi-Node-version monorepos.** A workspace
  where the frontend wants Node 22, a legacy Lambda wants
  Node 18, and a build script wants Node 20 just works:
  each subdirectory carries a `.node-version`, `cd` does the
  right thing, no ceremony.
- **CI runners that boot fast.** GitHub Actions / GitLab CI
  jobs that need a specific Node minor see `fnm install
  --install-if-missing` complete in seconds, against a fresh
  cache, with no Python/curl/bash dependency tree to fight.
- **Dev containers and Codespaces.** `fnm`'s static binary
  drops into a minimal Alpine / Debian-slim image without
  pulling `bash` or `git` as transitive deps the way a
  `nvm`-based image would.
- **Reducing shell startup tax.** Anyone whose `~/.zshrc`
  has crept past 800 ms because of `nvm` initialization
  reclaims most of that with one swap.

## Why pick this

Pick `fnm` when the question is "I need `nvm`'s feature set
without `nvm`'s shell-startup cost, and I want one binary I
can drop into a Dockerfile / CI image / fresh laptop with no
bash-script supply chain." It is the right tool for teams
standardising on per-project Node pinning via
`.node-version`, especially across Linux + macOS + Windows
contributors (Windows is a first-class target via PowerShell
+ winget). It composes naturally with [`mise`](../mise/),
[`asdf`](../asdf/) (use `mise` if you also need
Python / Ruby / Go version management in one tool, use `fnm`
when Node is the only language you need to multi-version).

## Comparable alternatives

- [`mise`](../mise/) — polyglot version manager (Node + Python
  + Ruby + Go + …) plus a task runner; pick when you need >1
  language pinned per project.
- [`volta`](../volta/) — Rust Node manager from LinkedIn with
  per-project automatic pinning baked into `package.json`; pick
  when you want toolchain pinning enforced at install time
  rather than via a separate version file.
- `nvm` — the original; pick only if you specifically want the
  bash-script behaviour and don't mind the startup cost.
- `n` — minimal Node manager from tj/n; smaller surface than
  `fnm` but no shell-aware auto-switching on `cd`.
- Corepack (built into Node 16.10+) — handles `pnpm`/`yarn`
  pinning but not Node itself; use alongside `fnm`.
