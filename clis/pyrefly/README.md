# pyrefly

> **Fast static type checker for Python, written in Rust by
> Meta.** Successor lineage to Pyre with a from-scratch
> bidirectional inference engine, an LSP server for editor
> integration, and IDE-grade incremental rechecking that
> handles million-line monorepos in seconds rather than
> minutes. Pinned to **0.63.1**
> ([LICENSE](https://github.com/facebook/pyrefly/blob/main/LICENSE),
> MIT; version checked via
> `gh api repos/facebook/pyrefly/releases/latest`).

Source: <https://github.com/facebook/pyrefly>

## TL;DR

`pyrefly` is the open-source replacement Meta is building for
its previous Python type checker. It accepts standard PEP 484 /
PEP 526 / PEP 604 / PEP 695 annotations, runs a bidirectional
type-inference pass (so unannotated locals still get checked),
and persists analysis state across runs so an incremental
recheck of a 1k-file project after a 10-line edit takes tens of
milliseconds, not the cold-start seconds `mypy` needs. The
binary doubles as a language server (`pyrefly lsp`) that VS
Code / Neovim / Helix can talk to via standard LSP for
hovers, completions, go-to-def, and inline diagnostics.

## Install

```bash
# Recommended: pip into the project venv so checks see your deps
pip install pyrefly

# Or one-shot via uv / pipx
uv tool install pyrefly
pipx install pyrefly

# Editor integration: any LSP-aware client can launch
#   pyrefly lsp --stdio
```

## Example

```bash
# Check the current project (auto-discovers pyproject.toml)
pyrefly check

# Check a specific package, treating warnings as errors for CI
pyrefly check --strict src/mypkg/

# Run as a long-lived language server for an editor
pyrefly lsp --stdio

# Print the inferred type of an expression at a location
pyrefly inspect src/mypkg/api.py:42:8
```

## When to use

- You have a large Python codebase where `mypy` cold start
  (or even `--cache-fine-grained` warm start) is the bottleneck
  in CI or in your editor.
- You want a single binary type checker that also serves as
  the LSP backend, removing the
  Pyright-Node-vs-mypy-Python split.
- You're starting a new project and want the fastest available
  feedback loop for PEP 695 generics, `Self` types, and
  `TypeIs` / `TypeGuard` narrowing.

## When NOT to use

- Your codebase depends on a `mypy` plugin (SQLAlchemy,
  attrs-pre-attribute-narrowing, Django stubs with
  `mypy_django_plugin`) — pyrefly does not implement the mypy
  plugin protocol; check plugin compatibility first.
- You need the exact warning catalog Pyright produces for an
  external contract (e.g., a library that publishes "passes
  Pyright strict") — pyrefly's diagnostics overlap heavily but
  are not byte-identical.
- You're on Python &lt; 3.9; pyrefly tracks modern syntax (PEP
  604 unions, PEP 695 generics) and is not optimized for
  legacy targets.

## Orthogonality vs existing zoo entries

- **vs [`ruff`](../ruff/)** — ruff is a *linter* and
  *formatter* (style, bug patterns, import sorting,
  Python-source rewrites). Pyrefly is a *type checker*
  (semantic analysis of `int | None` flowing through your
  program). They answer disjoint questions and the
  recommended setup runs both: ruff in pre-commit for
  style/lints, pyrefly in CI/editor for types.
- **vs [`ty`](../ty/)** — `ty` is Astral's in-development
  type checker (also Rust, also fast); pyrefly is Meta's.
  Both are alpha-quality and racing to feature parity with
  mypy + Pyright. Pick whichever passes against your codebase
  today; the loser will probably converge on the winner's
  diagnostics.
- **vs [`pydantic-ai`](../pydantic-ai/) /
  [`instructor`](../instructor/)** — those use `pydantic` for
  *runtime* schema validation. Pyrefly checks types
  *statically* before the program runs; a project will
  typically use both layers.
- **vs [`sqruff`](../sqruff/) / [`ruff`](../ruff/)** — same
  "Rust binary, Python-friendly install path, sub-second
  startup, LSP-capable" pattern, applied to a different layer
  of the Python toolchain (semantic types vs lexical lints).

## Niche / tags

`python` · `type-checker` · `static-analysis` · `lsp` ·
`rust` · `incremental` · `meta`
