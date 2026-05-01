# pipx

## What it does
A **per-application Python installer** that puts each CLI tool into
its own isolated virtualenv under `~/.local/pipx/venvs/<app>/` and
exposes the entry-point scripts on `$PATH` via `~/.local/bin/`. `pipx
install ruff` gives you the `ruff` binary without touching the system
Python or the active project venv; `pipx upgrade-all` updates every
managed app in one pass; `pipx run black .` runs a tool ephemerally
without installing it at all (cached venv, auto-discarded). Each app
gets its own dependency tree, so you can have `pipx install httpie`
and `pipx install awscli` coexist even when their `requests` /
`urllib3` pins disagree — the canonical "the user-level CLI tool
manager for Python" project, maintained under the `pypa` org.

## Why it's interesting
Different shape from `pip install --user` (drops every package's deps
into one shared `~/.local/lib/python3.x/site-packages/` — first
version conflict bricks your whole user site), from `uv tool install`
(uv now ships an explicitly pipx-shaped `tool` subcommand: pick uv if
you already use it as your installer / resolver and want one binary
to do everything; pick pipx for the smaller, focused, longstanding
tool with stable semantics and a `run` subcommand for ephemeral one-
shot execution), from `pyenv` / `mise` / `asdf` (those manage Python
*interpreters*, not per-app venvs), from `condax` (conda equivalent
of pipx — same shape, conda ecosystem instead of PyPI), and from
distro packages (slow to update, often a major version behind). pipx
is the *isolate every Python CLI in its own venv with one command*
shape: pick it specifically when you want `black` / `ruff` / `httpie`
/ `poetry` / `awscli` / `aws-sam-cli` / `youtube-dl` / `pre-commit`
on `$PATH` without dependency conflicts and without committing to uv
as your full toolchain. Do **not** pick it for *libraries* you import
in code (use `pip` / `uv pip` into a project venv), or for anything
that is not Python (use `npx` / `cargo install` / `gh extension`).

## Niche category
Per-application Python virtualenv installer — the user-level CLI
tool manager for the PyPI ecosystem.

## Repo
https://github.com/pypa/pipx

## Version pinned
`1.11.1` (latest tagged release per
`gh api /repos/pypa/pipx/releases/latest`)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install pipx
pipx ensurepath

# pip (any platform — bootstraps from the system / user Python)
python3 -m pip install --user pipx
python3 -m pipx ensurepath

# apt / dnf / pacman packages also exist on most distros
```

## Usage examples
```sh
# Install a CLI tool into its own venv and put it on $PATH
pipx install ruff
pipx install httpie
pipx install awscli

# List every installed app + its venv + its Python version
pipx list

# Upgrade everything in one pass
pipx upgrade-all

# Run a tool ephemerally, no install (venv cached for reuse)
pipx run black --check .
pipx run --spec 'cookiecutter==2.5.0' cookiecutter gh:foo/template

# Inject an extra dep into an existing managed app's venv
pipx inject mypy types-requests types-PyYAML

# Pin a specific Python interpreter for one app
pipx install --python python3.11 ansible

# Reinstall everything against a newer Python (e.g. after upgrading 3.11 -> 3.12)
pipx reinstall-all --python python3.12
```
