# uv

- **Repo:** https://github.com/astral-sh/uv
- **Version:** 0.11.8 (2026-04-27)
- **License:** MIT or Apache-2.0 ([LICENSE-MIT](https://github.com/astral-sh/uv/blob/main/LICENSE-MIT), [LICENSE-APACHE](https://github.com/astral-sh/uv/blob/main/LICENSE-APACHE))
- **Language:** Rust
- **Install:** `brew install uv` · `curl -LsSf https://astral.sh/uv/install.sh | sh` · `pipx install uv` · `cargo install --locked uv` · prebuilt binaries on the GitHub release page · binary name is `uv`

## What it does

`uv` is an end-to-end Python package and project manager written in
Rust by the same team behind `ruff`. It collapses what used to be
five separate tools — `pip`, `pip-tools`, `virtualenv`, `pyenv`,
and `pipx` — into a single static binary with a unified UX and a
shared on-disk cache. Concretely, `uv` will: install and pin
arbitrary CPython / PyPy versions (`uv python install 3.13`),
create and manage per-project virtual envs without ever calling
`python -m venv`, resolve dependency graphs an order of magnitude
faster than `pip-tools` thanks to a parallel resolver and
content-addressed wheel cache, generate cross-platform lockfiles
(`uv.lock`) that are reproducible across Linux / macOS / Windows in
one file, and run one-off scripts with inline PEP 723 dependency
declarations (`uv run script.py` reads the `# /// script` block at
the top and materialises the env on the fly).

It also speaks `pip`-compatible CLI surface (`uv pip install …`,
`uv pip compile …`) so existing scripts and Dockerfiles port with
near-zero friction, and it ships a `pipx`-shaped tool runner
(`uv tool install ruff`, `uvx ruff check`) for installing CLIs into
isolated envs.

## When to pick it / when not to

Reach for `uv` when you want **the fastest, least-fragile Python
toolchain that exists today**. It is the right answer for new
projects, for CI pipelines that currently spend minutes on `pip
install`, for monorepos with multiple Python services that need a
shared lockfile, and for ML / agent codebases where dependency
churn is constant and resolver speed dominates iteration time.
The `uv run` + PEP 723 inline-deps flow is also ideal for
LLM-generated one-off scripts that would otherwise litter your
disk with venvs.

Skip it when your project's binary deps live in **conda-forge**
(CUDA toolkits, GDAL, BLAS pinned against a specific MKL build) —
[`pixi`](../pixi/) is the right tool there because it reuses the
conda-forge ecosystem. Also skip if you absolutely require Poetry's
`pyproject.toml`-driven plugin ecosystem; `uv` reads `pyproject.toml`
but does not run Poetry plugins.

## Why it matters in an AI-native workflow

Agent loops generate and tear down throwaway Python envs constantly
— for evaluating a generated patch, for running a tool dependency,
for sandboxed code execution. With `pip`/`venv` each cycle costs
seconds-to-minutes and the cache invariably bloats. `uv`'s shared
hardlinked cache plus parallel resolver brings env creation under a
second on a warm cache, which turns "spin a fresh env per tool
call" from infeasible to default. The PEP 723 inline-script flow
also gives an LLM a single artifact (one `.py` file with a
front-matter block) that captures both code and deps — much
easier to round-trip than a directory tree.

## Example invocations

```bash
# Bootstrap a project with a pinned Python
uv init my-agent && cd my-agent
uv python pin 3.13

# Add deps; updates pyproject.toml + uv.lock atomically
uv add httpx pydantic 'fastapi[standard]'
uv add --dev pytest ruff

# Create / sync the venv from the lockfile
uv sync

# Run anything inside the project env without activating
uv run python -m my_agent
uv run pytest

# One-off script with inline deps (PEP 723)
uv run --with rich --with httpx scratch.py

# Install a CLI tool into an isolated env (pipx-shaped)
uv tool install ruff
uvx ruff check .

# pip-compatible surface for legacy workflows
uv pip install -r requirements.txt
uv pip compile pyproject.toml -o requirements.lock
```

## Alternatives in this catalog

- [`pixi`](../pixi/) — conda-forge-based package manager; pick pixi
  when system libs (CUDA, GDAL) dominate, uv when pure-PyPI is fine.
- [`mise`](../mise/) — multi-language runtime manager (Node + Python
  + Go + Ruby); pick mise when Python is one of many, uv when
  Python is the project's whole world.
- [`fnm`](../fnm/) — sibling story for Node version management;
  uv is to Python what fnm is to Node.
