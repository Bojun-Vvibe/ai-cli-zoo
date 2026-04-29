# ruff

> **An extremely fast Python linter and code formatter, written in
> Rust** — one ~25 MB binary that replaces Flake8 + isort + Black +
> pydocstyle + pyupgrade + autoflake + bandit (subset) + ~50 other
> Python lint plugins, runs 10–100× faster than the equivalent
> pip-installed pipeline, and applies safe autofixes for hundreds of
> rules. Pinned to **0.15.12** (released 2026-04-24),
> [LICENSE](https://github.com/astral-sh/ruff/blob/main/LICENSE) (MIT).

Source: <https://github.com/astral-sh/ruff>

## TL;DR

`ruff` is what you reach for when your Python repo's pre-commit
hook chain (`black` → `isort` → `flake8` → `pyupgrade` →
`pydocstyle`) takes 30 seconds on a small change and you have
finally accepted that the right answer is "one Rust binary that
does all of it in 200 ms". `ruff check` lints, `ruff format` is a
Black-compatible formatter (95%+ identical output), `ruff check
--fix` autofixes hundreds of rules in place, and the rule set
covers the union of Flake8 + its top ~25 plugins (bugbear,
comprehensions, simplify, pyupgrade, pep8-naming, pydocstyle,
bandit security subset, NumPy / pandas style, async, perf, …).
Configuration is one `[tool.ruff]` block in `pyproject.toml`;
selecting rules is `select = ["E", "F", "I", "B", "UP"]` etc.,
and the rule registry is searchable at runtime with `ruff rule
<code>`.

## Install

```bash
# uv (recommended — same vendor)
uv tool install ruff

# pipx
pipx install ruff

# pip
pip install ruff

# Homebrew (macOS / Linux)
brew install ruff

# Cargo
cargo install --locked ruff

# pre-commit hook
# - repo: https://github.com/astral-sh/ruff-pre-commit
#   rev: v0.15.12
#   hooks:
#     - id: ruff
#     - id: ruff-format

# verify
ruff --version    # ruff 0.15.12
```

## Why catalogued

`ruff` has effectively become the default Python lint + format
toolchain for any project started after ~2024, and AI coding
agents that scaffold or edit Python code (aider, claude-code,
opencode, codex, gemini-cli, crush) increasingly invoke
`ruff check --fix && ruff format` as the canonical
post-edit cleanup step instead of the slower black + isort +
flake8 trio. Speed matters here because the agent loop runs
the linter per turn — a 200 ms `ruff check` is acceptable in a
tight edit cycle in a way a 15 s `flake8` on a large repo is
not. The Black-compatible formatter also eliminates a class of
"agent edit triggers a format-only diff that breaks the next
review" failure modes, and the autofix surface (hundreds of
rules) lets the agent rely on `ruff --fix` to clean up its own
output rather than emitting more LLM tokens to do the same job.

## When to pick

- You are starting a new Python project, full stop — there is
  no longer a credible reason to assemble flake8 + isort +
  black manually.
- Your existing flake8 + black + isort pipeline is too slow on
  the repo's hot path (pre-commit, editor save, agent loop, CI
  PR check).
- An AI coding agent is editing Python in your repo and you
  want a fast, deterministic post-edit cleanup gate.
- You want one config block (`[tool.ruff]` in `pyproject.toml`)
  instead of `setup.cfg` + `.flake8` + `pyproject.toml` +
  `.isort.cfg`.
- You want autofix for hundreds of rules without composing
  `autoflake` + `pyupgrade` + `isort` separately.

## When to skip

- You depend on a Flake8 plugin that has no `ruff` equivalent
  and rewriting it as a `ruff` rule (Rust) is not on your
  team's roadmap (rare — check `ruff rule --all` first).
- You need pylint's deeper semantic analysis (type-aware
  refactors, design-smell detection like too-many-instance-
  attributes); `ruff` deliberately stays in the lint-not-
  type-check lane (use [`ty`](https://github.com/astral-sh/ty)
  or `mypy` / `pyright` for typing).
- Your team has a heavily customised Black config that you
  cannot port to `ruff format`'s near-identical-but-not-bit-
  exact output (uncommon, but check the diff before flipping).
- You need lint coverage of non-Python files (TOML, YAML,
  Markdown) — `ruff` is Python-only.

## Composes well with

- [`uv`](../uv/) — same vendor (Astral), same Rust speed
  philosophy; `uv tool install ruff` and `uv run ruff check`
  is the canonical pairing.
- `ty` / `mypy` / `pyright` — `ruff` does not type-check;
  pair with one of these for the typing layer.
- Pre-commit (`ruff-pre-commit`) — official hook repo, replaces
  the black + isort + flake8 hooks with two entries.
- AI coding CLIs in this catalog ([`aider`](../aider/),
  [`claude-code`](../claude-code/), [`opencode`](../opencode/),
  [`codex`](../codex/), [`crush`](../crush/)) — add
  `ruff check --fix && ruff format` as the post-edit hook so
  the agent's diffs land lint-clean without burning extra
  model tokens.
