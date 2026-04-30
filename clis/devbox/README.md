# devbox

> **Per-project isolated dev shells backed by Nix, without
> writing Nix.** A `devbox.json` declares packages by name
> (`python@3.12`, `nodejs@20`, `postgresql@16`); `devbox shell`
> drops you into a hermetic shell where exactly those versions
> are on `PATH`, reproducible across macOS / Linux / WSL and
> across teammates without polluting the host. Pinned to
> **0.17.2**
> ([LICENSE](https://github.com/jetify-com/devbox/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/jetify-com/devbox>

## TL;DR

`devbox add python@3.12 nodejs@20` writes the package + its
exact Nix store hash into `devbox.json` + `devbox.lock`;
`devbox shell` materialises that environment in seconds with
zero global install. Init-hook scripts (`init_hook`), per-project
scripts (`devbox run test`), service definitions
(`devbox services up postgres`), and a generated `Dockerfile`
(`devbox generate dockerfile`) all consume the same lockfile,
so "works on my machine" collapses to "works in the same
hermetic closure on every machine". No Nix knowledge required —
the package search is keyword-based against the Nixpkgs
catalogue at <https://www.nixhub.io>.

## Install

```bash
# Install script (default, installs to /usr/local/bin)
curl -fsSL https://get.jetify.com/devbox | bash

# Homebrew
brew install jetify-com/devbox/devbox

# Container (CI-friendly)
docker run --rm -v $(pwd):/code -w /code jetpackio/devbox:0.17.2 shell -- run test
```

## Example

```bash
# Bootstrap a project
devbox init
devbox add python@3.12 nodejs@20 postgresql@16 redis

# Enter the hermetic shell (first run downloads + caches Nix store paths)
devbox shell

# One-shot run without entering the shell
devbox run -- python -m pytest

# Background services declared in devbox.json
devbox services up
devbox services ls

# Reproduce the env as a container image for CI
devbox generate dockerfile
docker build -t myapp:devbox .
```

## When to use

- Your team wants Nix-grade reproducibility but cannot afford a
  Nix learning-curve tax across every contributor.
- You need the same `python@3.12 + postgres@16 + redis` stack
  on a Mac laptop, a Linux CI runner, and a Codespace, with one
  lockfile.
- You want per-project tool versions without `asdf` /
  `mise` / `pyenv` / `nvm` collisions on `PATH`.

## When NOT to use

- You already run Nix flakes natively and want full Nix
  expressiveness — devbox intentionally hides Nix and is a step
  back in flexibility.
- The project is single-language and a language-native version
  manager (`uv`, `pnpm`, `cargo`) already pins everything you
  need.
- You need full container-level isolation (kernel, network,
  filesystem) — devbox is a shell environment, not a sandbox;
  reach for `docker` or `container-use`.
