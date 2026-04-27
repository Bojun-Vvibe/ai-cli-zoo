# tach

> **A Python tool to visualize and enforce module dependencies** — a
> Rust-implemented `pip install`-shaped CLI that parses a Python
> codebase into a module graph, declares allowed inter-module edges
> in a single `tach.toml`, and fails CI on undeclared imports so a
> growing Python project gets the layered-architecture hygiene that
> Java / Go / Rust have by default. Pinned to **v0.34.1** (commit
> `28569d65fe42670b3eb45d58f16427ba2677a8b8`,
> [LICENSE](https://github.com/gauge-sh/tach/blob/main/LICENSE),
> MIT).

Source: <https://github.com/gauge-sh/tach>

## TL;DR

`tach` is "import-linter, but Rust-fast and adoptable in one
afternoon". `tach mod` walks the project and writes a `tach.toml`
listing every top-level module; `tach sync` infers the current
dependency edges and writes them as the *baseline* allowed set;
`tach check` parses every Python file, resolves every `import`,
and fails (non-zero exit, one finding per offending line) if a
module imports another module not on its `depends_on` list. Edges
are declared in TOML, not docstring DSL, so the rule file diffs
cleanly in PRs and a "module X may now depend on module Y" change
is one line for review. The Rust core (PyO3-bound) parses tens of
thousands of files per second so the check is cheap enough for
every PR run, and the same binary supports `tach show` (Mermaid /
JSON / D2 graph export for documentation), `tach report` (per-module
incoming / outgoing dependency tables), and `tach test` (run only
the tests downstream of the files changed in the current diff —
the Python answer to `bazel query`).

## Install

```bash
# uv (recommended)
uv tool install tach

# pipx
pipx install tach

# pip
pip install tach

# verify
tach --version

# bootstrap a project
cd my-python-repo
tach mod              # write tach.toml with module list
tach sync             # populate the allowed-edges baseline
tach check            # exit non-zero on undeclared imports
```

Single binary distributed via PyPI wheels (Rust core, Python CLI
wrapper). No runtime impact — `tach` is a static analyzer, not an
import hook.

## One Concrete Example

A medium Python service `app/` with `api/`, `services/`, `db/`,
and `vendor/` modules:

```bash
# 1. one-time bootstrap
cd app
tach mod         # writes tach.toml listing api/, services/, db/, vendor/
tach sync        # records the current dependency edges as the baseline
                 # (api -> services, services -> db, db -> vendor)

# 2. someone adds `from app.db.models import User` to api/handlers.py
#    (api skipping straight past the services tier)
tach check
# api/handlers.py:3: ImportError: 'app.db.models' is not allowed
#   from 'app.api.handlers'. Add 'db' to the depends_on list of 'api'
#   in tach.toml, or route the call through 'services'.
# Exit 1.

# 3. visualise current allowed graph
tach show --format mermaid > docs/architecture.mmd

# 4. only run tests downstream of the diff (CI optimisation)
tach test --base origin/main
# pytest invocation, scoped to the modules transitively affected
# by the diff; cuts a 12-minute test suite to ~90s on small PRs
```

## Niche It Fills

**The "import-linter that you actually adopt" for Python codebases.**
Python has no built-in module visibility — every `import` is
allowed, layered architectures degrade silently into a ball of
mutual references over a year of growth, and the existing
`import-linter` has the right idea but needs a hand-authored
contract DSL that few teams ever write. `tach` inverts the
adoption flow: the *current* graph becomes the baseline in one
command, and the rule is "no new edges without a `tach.toml`
change". For a team that wants Java / Go / Rust-style module
boundaries without rewriting the codebase, the on-ramp is one
afternoon, and the Rust core makes it cheap enough that every PR
runs the check.

## Vs Already Cataloged

- **Vs [`ast-grep`](../ast-grep/):** `ast-grep` is a generic
  multi-language structural-search-and-replace tool; `tach` is a
  Python-specific dependency-graph linter. Use `ast-grep` to find
  / rewrite specific code patterns; use `tach` to enforce
  architectural layering.
- **Vs [`grep-ast`](../grep-ast/) / [`symbex`](../symbex/):** those
  are search tools for finding symbols across a codebase; `tach`
  is an enforcement tool for *which symbols may import which*.
  They compose naturally: search with the others, enforce with
  `tach`.
- **Vs `import-linter`:** same problem space, different ergonomics.
  `import-linter` requires hand-writing a contract spec before you
  see any value; `tach sync` infers the current graph and treats
  it as the rule, so the friction of starting is one command.

## Caveats

- Static analysis only — `tach` does not see dynamic imports
  (`importlib.import_module(name)` with a runtime-built name,
  `__import__(...)` inside a function, monkey-patched re-exports).
  For codebases that lean on dynamic imports, expect false
  negatives; pair with runtime monitoring if dynamic imports are
  load-bearing for a real boundary.
- The default `tach mod` granularity is **top-level modules**;
  enforcing "this submodule may not import that submodule" needs
  explicit nested module entries in `tach.toml` (the docs cover
  the recursion pattern, but it is opt-in).
- `tach test --base <ref>` requires git history available in CI;
  shallow clones (`git clone --depth=1`) defeat it. Configure
  CI to fetch the merge base.
- The on-disk format is stable within `0.x` but the project is
  pre-1.0 — pin the version in `pyproject.toml` and review
  release notes before bumping minors.
