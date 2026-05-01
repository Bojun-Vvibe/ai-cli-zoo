# pyenv

> **Per-shell Python version switcher built from a `shims/` PATH
> directory and `.python-version` file lookup that walks up parents
> from `$PWD` — the boring, decade-stable foundation under every
> "managed Python environment" in the catalog (`uv`, `rye`, `pdm`,
> `pixi`, `mise`).** Pinned to **v2.6.28**, MIT
> ([LICENSE](https://github.com/pyenv/pyenv/blob/master/LICENSE)).

- **Repo:** https://github.com/pyenv/pyenv
- **Latest version:** v2.6.28
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `language-runtime` / `version-manager`
- **Language:** Bash (+ optional `python-build` plugin in Bash)

## What it does

`pyenv` does one thing well: switch the active Python version per
shell, per directory, or per user, by sticking a `shims/` directory
at the front of `PATH` whose `python` / `pip` / `python3.12`
executables are tiny shell scripts that re-resolve to the right
real interpreter at every call. Resolution order is `PYENV_VERSION`
env var → nearest `.python-version` file walking up from `$PWD` →
user `~/.pyenv/version` → system. Versions are installed by
`pyenv install 3.12.7` which downloads and compiles CPython (or
PyPy / Stackless / Jython / GraalPy / MicroPython / pyston / etc.)
from source via the `python-build` plugin, with the build cache at
`~/.pyenv/cache/`. The shim approach means *every* tool that finds
Python through `PATH` — your editor, your test runner, your
`#!/usr/bin/env python` script, a `Makefile` invoking `python -m
pip`, a docker `RUN` line — sees the same active version with no
per-tool config. `pyenv-virtualenv` (separate plugin) layers
virtualenv creation onto the same shim mechanism so
`pyenv virtualenv 3.12.7 myproj && pyenv local myproj` makes a
named env auto-activate when you `cd` in.

## Install

```sh
# macOS
brew install pyenv

# Linux (any distro) — installer script
curl -fsSL https://pyenv.run | bash

# Then add to shell init (zsh shown)
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - zsh)"
```

## Usage

```sh
# Install and pin a Python version for this directory
pyenv install 3.12.7
pyenv local 3.12.7        # writes .python-version, auto-activates on cd

# Switch globally for all shells without a .python-version
pyenv global 3.11.10 3.12.7   # 3.11 default; 3.12 also on PATH as python3.12
```

## Use when

- You want a **single mechanism** that works across every Python
  tool, IDE, and script that resolves `python` via `PATH`.
- You need build-from-source CPython with custom `--enable-optimizations`
  / `--with-openssl=...` flags (set via `PYTHON_CONFIGURE_OPTS`).
- You want `.python-version` files committed to repos so collaborators
  get the same interpreter without reading docs.

## Skip when

- You're already using [`uv`](../uv/), [`rye`](../rye/),
  [`pdm`](../pdm/), [`pixi`](../pixi/), or [`mise`](../mise/) — they
  bundle their own Python download path (often the prebuilt
  `python-build-standalone` releases) and a shim-or-PATH activation,
  making `pyenv` redundant.
- You need fast first-use install — `pyenv install` compiles CPython
  from source (~3-10 min); `uv python install 3.12` pulls a prebuilt
  in seconds.
- You're on Windows native (use `pyenv-win` fork or `uv` instead).

## Comparison to nearest neighbours

- **vs [`uv`](../uv/):** `uv python install` uses prebuilt
  `python-build-standalone` binaries (seconds, no compiler needed)
  and integrates with `uv venv` / `uv pip` / `uv run`; `pyenv` builds
  from source (slower, but supports any custom configure flag).
  Pick `uv` for greenfield Python work; pick `pyenv` when you need
  the source-build escape hatch or already have a `.python-version`
  ecosystem.
- **vs [`mise`](../mise/):** `mise` is `pyenv` + `nodenv` + `rbenv`
  + `tfenv` + ~50 others in one Rust binary, with `.tool-versions` /
  `.mise.toml` config. Pick `mise` when you manage 3+ language
  runtimes; pick `pyenv` when Python is the only one and you want
  the focused, decade-tested tool.
- **vs [`asdf`](https://asdf-vm.com/):** `asdf` is the polyglot
  predecessor of `mise` (Bash + plugins); same trade-off — `pyenv`
  is faster and more correct for Python specifically because it
  isn't generalised over a plugin protocol.
