# rye

> **An all-in-one Python project & toolchain manager** — manages
> Python interpreters, virtualenvs, dependencies, lockfiles, and
> packaging from one binary, with a `cargo`-shaped CLI. Originally
> by Armin Ronacher; now stewarded by the [`uv`](../uv/) team at
> Astral as its experimental sibling and gradual migration target.
> Pinned to **v0.44.0**
> ([LICENSE](https://github.com/astral-sh/rye/blob/main/LICENSE),
> MIT).

Source: <https://github.com/astral-sh/rye>

## TL;DR

`rye` is what you reach for when you want one tool to do for Python
what `cargo` does for Rust: install the interpreter, create the
project, manage the virtualenv, resolve dependencies, write the
lockfile, run scripts, and build a wheel — all from a single
binary, with no `pyenv` + `pipx` + `pip-tools` + `poetry` +
`virtualenv` stack to assemble. It downloads pre-built portable
CPython interpreters (no system Python required, no `pyenv`
compile step), creates a `.venv` per project, uses `uv` under the
hood as the resolver/installer (so installs are 10–100× faster
than pip), and writes a `pyproject.toml` + `requirements.lock` +
`requirements-dev.lock` triplet that is fully reproducible across
machines.

## Install

```bash
# official installer (downloads the rye binary + bootstraps ~/.rye)
curl -sSf https://rye.astral.sh/get | bash

# Homebrew (macOS / Linux)
brew install rye

# direct binary
curl -L https://github.com/astral-sh/rye/releases/download/0.44.0/rye-aarch64-macos.gz \
    | gunzip > /usr/local/bin/rye && chmod +x /usr/local/bin/rye

# add the shim dir to PATH (the installer writes this to your shell rc)
source "$HOME/.rye/env"

# verify
rye --version    # rye 0.44.0
rye toolchain list
```

## License

MIT — see
[LICENSE](https://github.com/astral-sh/rye/blob/main/LICENSE).
Permissive, no copyleft, vendor freely. Same license shape as the
rest of the Astral toolchain ([`uv`](../uv/),
[`ruff`](../ruff/)).

## One Concrete Example

```bash
# 1. start a new project (creates pyproject.toml, src/, .venv/,
#    .python-version, README, .gitignore — all the right shapes)
rye init my-service && cd my-service

# 2. pin the interpreter (downloads a portable CPython if not cached)
rye pin 3.12.7

# 3. add runtime + dev dependencies (writes pyproject.toml, resolves,
#    installs into .venv, updates the lockfile)
rye add httpx pydantic
rye add --dev pytest mypy ruff

# 4. sync (idempotent reconcile of .venv against the lockfile —
#    what CI runs)
rye sync

# 5. run a tool from the project venv (no `source .venv/bin/activate`
#    needed; rye injects PATH via shims)
rye run pytest -q
rye run python -m my_service

# 6. install a global CLI tool (rye-managed, isolated venv, like pipx)
rye tools install ruff
rye tools install httpie

# 7. build a wheel + sdist for PyPI
rye build && rye publish

# 8. lock without installing (CI prep, generates fresh lockfiles)
rye lock --update-all
```

## Niche It Fills

**Collapse the Python toolchain — `pyenv` + `pipx` + `virtualenv`
+ `pip-tools` + `poetry` + `setuptools` + `twine` — into one
`cargo`-shaped binary.** A typical Python developer in 2025 has
six tools doing what `cargo` does alone: `pyenv` to pick the
interpreter, `python -m venv` (or `virtualenv`) to make the env,
`pip install` to install, `pip-compile` to lock, `pipx` for global
CLIs, `build` + `twine` to ship. Each has its own install method,
its own config file, its own corner cases (`pyenv` needs build
dependencies, `pipx` needs its own `~/.local`, `pip-compile` and
`poetry` disagree about lockfile shape). `rye` is one binary that
covers all of it, with a CLI that mirrors `cargo` (`rye init`,
`rye add`, `rye sync`, `rye run`, `rye build`, `rye publish`).

## Why use it

Three things `rye` does that the legacy toolchain stack does not:

1. **Manages the interpreter, not just the env.** `rye pin 3.12.7`
   downloads a portable, statically-linked CPython build (from the
   [`python-build-standalone`](https://github.com/astral-sh/python-build-standalone)
   project) into `~/.rye/py/`, no system Python and no `pyenv`
   compile step required. Switching a project from 3.11 to 3.12
   is `rye pin 3.12 && rye sync` — no global state mutation, no
   `pyenv local` shim drift, the interpreter version lives in the
   committed `.python-version` file. CI gets the exact same
   interpreter byte-for-byte that the developer used.
2. **`uv`-backed resolver and installer.** Under the hood `rye`
   uses [`uv`](../uv/) for dependency resolution and wheel
   installation, which means `rye sync` on a 200-package
   `requirements.lock` runs in a few seconds where `pip install
   -r requirements.txt` takes 30–60. The lockfile is a flat
   `requirements.lock` (just the resolved set with hashes), which
   is greppable, diffable, and reproducible — no
   `poetry.lock`-shaped opaque tree.
3. **One CLI surface for project + tool + interpreter.** `rye add`
   manages project deps, `rye tools install` manages global CLIs
   (in their own isolated venvs, like `pipx` but without the
   second tool), `rye toolchain` manages interpreters, `rye
   build` / `rye publish` ships to PyPI. There is no
   "which tool owns this command" decision tree — everything
   project-related is `rye <verb>`.

For an LLM-CLI workflow that scaffolds Python services, `rye init
+ rye add` produces a buildable, lockfile-clean project in one
shot, and the resulting `pyproject.toml` is standard PEP 621 so
non-`rye` users can `pip install -e .` and continue.

## Vs Already Cataloged

- **Vs [`uv`](../uv/):** complementary and converging. `uv` is
  the lower-level Rust resolver/installer/venv-manager from the
  same team (Astral); `rye` is the higher-level "one binary for
  the whole Python project lifecycle" wrapper. Astral's published
  direction: `uv` is the strategic long-term tool (it has
  absorbed `rye`'s project-management features into `uv` itself
  with `uv init`, `uv add`, `uv run`, `uv lock`), and `rye` is in
  maintenance / migration-target mode. Choose `uv` for new
  projects today; choose `rye` if you already have a `rye`
  project, want the slightly more opinionated CLI shape, or are
  comparing the two before committing.
- **Vs [`pixi`](../pixi/):** sibling tool, conda-ecosystem
  flavored. `pixi` is `rye`-shaped but built on the conda /
  conda-forge package universe, so it can manage non-Python
  binaries (CUDA toolkits, FFmpeg, system libraries) alongside
  Python deps in the same lockfile. Pick `pixi` when your
  project needs binary scientific dependencies; pick `rye` (or
  `uv`) when pure-PyPI is enough.
- **Vs [`mise`](../mise/):** different layer. `mise` is a polyglot
  runtime version manager (`asdf`-shaped, manages Node + Python +
  Ruby + Go versions per directory). `rye` is Python-specific and
  also handles dependencies, lockfiles, and packaging. They
  compose: `mise` to manage the `rye` binary itself across
  machines, `rye` to manage projects.
- **Vs `poetry` (not cataloged):** both are project managers.
  `poetry` invented much of the modern Python project-manager UX
  but is Python-implemented (so it is slow on cold installs and
  cannot manage the interpreter), uses its own lockfile format,
  and historically resisted PEP 621 standardization. `rye` is
  Rust-binary fast, manages the interpreter, and writes
  PEP 621-compliant `pyproject.toml`. Migration: `rye` can
  consume an existing `poetry`-managed `pyproject.toml` with
  manual cleanup.

## Caveats

- **Officially in maintenance / migration mode.** Astral is
  consolidating Python project-management into [`uv`](../uv/);
  `rye`'s README explicitly recommends new projects start with
  `uv` and existing `rye` users migrate when convenient. `rye`
  still receives bug fixes and tracks `uv` versions, but
  greenfield work should default to `uv`.
- **Portable CPython is not your distro's CPython.** `rye`'s
  interpreters come from `python-build-standalone`, which patches
  CPython slightly for relocatability. 99 % of pure-Python and
  most C-extension code works identically; very low-level code
  that hard-codes interpreter paths or links against system
  libpython can hit edge cases. If your project links against
  `libpython.so` directly (rare — embedding scenarios), use the
  system interpreter via `rye pin --relaxed` and a system path.
- **Lockfile is `requirements.lock`, not `pyproject.toml`-native.**
  The lockfile sits next to `pyproject.toml` as a separate file
  with `pip`-style syntax + hashes. That is reproducible and
  greppable but not yet a PEP standard, so other tools (poetry,
  pdm) cannot consume it directly. Cross-tool compatibility lives
  at the `pyproject.toml` layer, not the lockfile layer.
- **Global tool installs share `~/.rye`.** Like `pipx`, `rye
  tools install` puts each global CLI in its own isolated venv
  under `~/.rye/tools/`. That is correct but means uninstalling
  `rye` itself (`rm -rf ~/.rye`) takes all installed CLIs with
  it. Back up the tool list (`rye tools list > tools.txt`) before
  major surgery.
- **Windows support is real but less battle-tested than
  macOS/Linux.** Most issues are around shim PATH handling on
  PowerShell vs CMD vs WSL. Linux containers and macOS
  developer machines are the well-trodden paths.
