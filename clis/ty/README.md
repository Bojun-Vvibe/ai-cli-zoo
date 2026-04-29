# ty

> Snapshot date: 2026-04. Upstream: <https://github.com/astral-sh/ty>

astral-sh's **extremely fast Python type checker and language
server**, written in Rust on top of the same Ruff infrastructure
(`ruff_python_ast`, `ruff_python_parser`, `red_knot` semantic
engine). It is the type-checker counterpart to what `ruff` did for
linting and `uv` did for resolution / install: a single static
binary that runs orders of magnitude faster than `mypy` /
`pyright`'s Node-based core, with the same `pyproject.toml`-shaped
configuration and a built-in LSP for editor integration.

It is deliberately pre-1.0 (`0.0.x` series) and the upstream
documents semantic gaps versus mypy / pyright; pick it today for
**speed on a known-working codebase**, not for the strictest
soundness guarantees.

## 1. Install

- `uv tool install ty` — recommended; same tooling family.
- `pipx install ty` or `pip install ty` — published to PyPI as
  `ty` with prebuilt wheels per platform.
- `brew install ty` (when the formula lands; check `brew search ty`).
- One-shot installer: `curl -LsSf https://astral.sh/ty/install.sh
  | sh` (Linux/macOS) / `irm https://astral.sh/ty/install.ps1 | iex`
  (Windows).
- Standalone tar.gz / zip per triple on the
  [releases page](https://github.com/astral-sh/ty/releases).
- Single static Rust binary, ~10 MB, no Python or Node runtime
  required to run the checker itself.

## 2. Version pin

**0.0.33** (released 2026-04-28). Verify with `ty --version`. The
`0.0.x` series ships breaking changes between minors — pin in CI.

## 3. License

MIT. SPDX: `MIT`. Full text at
[`LICENSE`](https://github.com/astral-sh/ty/blob/main/LICENSE) in
the repository root.

## 4. What it does

Two surfaces in one binary:

- **`ty check`** — the type-checker CLI. Walks a project, parses
  every `.py` / `.pyi` file with the Ruff parser, runs the `red_knot`
  semantic engine to infer and check types, and prints diagnostics
  with rich source context. Honours `pyproject.toml` (`[tool.ty]`),
  PEP 561 stub packages, the `typeshed` bundled stdlib stubs, and
  per-file `# ty: ignore[rule]` suppressions.
- **`ty server`** — a Language Server Protocol server consumed by
  `ty-lsp` / VS Code / Neovim / Helix / Zed for in-editor inlay
  hints, hover types, go-to-definition on type names, and live
  diagnostics on save.

Notable subcommands and flags:

- `ty check [paths]` — check a project / file / directory.
- `ty check --output-format=json | concise | full` — JSON for CI
  pipelines, concise for terminals, full with source spans for
  humans.
- `ty check --error-on-warning` — promote warnings to errors for
  CI gating.
- `ty version` / `ty --version` — print the build version.

## 5. Install & first use

```bash
uv tool install ty
ty --version

# check a project
cd my-python-project
ty check                            # honours pyproject.toml
ty check src/ tests/                # explicit paths
ty check --output-format=json > diagnostics.json

# editor LSP (example: Neovim via lspconfig)
:lua require('lspconfig').ty.setup{}
```

## 6. Example: minimal `pyproject.toml`

```toml
[tool.ty]
# project root inference handles the rest, but you can pin:
src = ["src", "tests"]
python-version = "3.12"

[tool.ty.rules]
# project-wide suppressions / promotions
unused-ignore-comment = "warn"
possibly-unresolved-reference = "error"
```

CI step:

```yaml
- name: ty
  run: uv tool run ty check --error-on-warning --output-format=concise
```

## 7. Why it matters in this catalog

`ty` is the type-checking peer of [`ruff`](../ruff/) and
[`uv`](../uv/) in the astral-sh single-binary Python toolchain — the
combination that has displaced the Python-ecosystem default of
"`mypy` + `flake8` + `black` + `isort` + `pip-tools`" with one Rust
binary per concern. For an LLM-generated Python codebase where the
agent emits dozens of files per run, sub-second whole-project
type-checking on every save / pre-commit means the type signal is
available inside the agent loop instead of after a five-minute mypy
pass; `ty check --output-format=json` is the structured diagnostic
stream a coding agent can ingest, classify, and self-repair from.

The pre-1.0 caveat is real: today, treat `ty` as the **fast
first-pass** checker that catches obvious type errors in the agent
loop, with `mypy --strict` or `pyright` as the slower, stricter
gate in CI. As `red_knot` matures, expect the strictness gap to
close.

## 8. Alternatives

- `mypy` — the reference Python type checker; slower, mature
  semantics, the one stub authors target.
- `pyright` — the TypeScript-implemented checker (Node runtime)
  shipped by the Pylance maintainers; strict-mode soundness;
  powers the Pylance editor extension.
- `pyre` — Meta's OCaml-implemented checker focused on large
  monorepos.
- `pytype` — Google's checker, type-inference heavy, slower.
- [`ruff`](../ruff/) — same astral-sh toolchain, lints not types;
  pair with `ty` rather than choose between them.

## 9. Telemetry

No analytics SDK in the binary. Outbound network access is limited
to the registry / PyPI fetches you trigger explicitly (e.g. when
resolving a stub package via `uv`); the checker itself runs fully
offline against the bundled typeshed stubs.
