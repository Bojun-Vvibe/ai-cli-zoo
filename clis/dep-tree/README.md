# dep-tree

> **dep-tree** — gabotechs/dep-tree, a single-binary tool that
> renders, queries, and lints the *file-level* import graph of a
> JavaScript / TypeScript / Python / Rust codebase. Pinned to
> **v0.23.4**, MIT — license file:
> [LICENSE](https://github.com/gabotechs/dep-tree/blob/main/LICENSE).

Source: <https://github.com/gabotechs/dep-tree>

## TL;DR

`dep-tree` parses your source tree, extracts every `import` /
`require` / `from … import` / `use` statement, and builds an
in-memory directed graph of file → file dependencies. It then does
three different jobs against that graph:

1. **Render it** — `dep-tree render src/index.ts` opens an
   interactive 3-D force-directed graph in the terminal (or the
   browser with `--browser`) so you can actually *see* the shape of
   the codebase, which files are central, and which subtrees are
   tangled.
2. **Query it** — `dep-tree entropy` computes a per-file "graph
   entropy" score that surfaces files importing wildly across
   unrelated subtrees, i.e. the natural refactoring targets.
3. **Lint it** — `dep-tree check` reads a `.dep-tree.yml` config
   that declares **allowed** and **disallowed** edges between
   directory globs (`web/* MUST NOT import server/*`,
   `domain/* MUST NOT import infra/*`) and exits non-zero on any
   violation. That last mode is the production use case: running
   `dep-tree check` in CI keeps a hexagonal / clean / layered
   architecture honest without having to wire ESLint plugins,
   `import/no-restricted-paths` arrays, or per-language equivalents.

It is one of the few tools that handles JS/TS *and* Python *and*
Rust under the same configuration grammar, so a polyglot monorepo
gets a single architecture-lint binary.

## Install

```bash
# Single static Go binary — releases are at
# https://github.com/gabotechs/dep-tree/releases/tag/v0.23.4

# Homebrew
brew install dep-tree

# Go
go install github.com/gabotechs/dep-tree@v0.23.4

# npm wrapper (downloads the right release binary on postinstall)
npm install -g dependency-tree-cli
```

## Example commands

```bash
# Interactive 3-D render of an entry-point's transitive imports
dep-tree render src/index.ts

# Same thing in the browser instead of the terminal
dep-tree render src/index.ts --browser

# Entropy report — files most likely to be the next refactor target
dep-tree entropy src/

# Architecture lint against a config file (CI mode)
dep-tree check
# reads .dep-tree.yml in repo root, exits non-zero on violations

# Explain why one file ends up importing another (debugging a
# disallowed-edge violation)
dep-tree explain src/web/page.tsx src/server/db.ts
```

A minimal `.dep-tree.yml` for a layered codebase:

```yaml
entrypoints:
  - src/index.ts
allow:
  - from: "src/web/**"
    to:   "src/domain/**"
  - from: "src/domain/**"
    to:   "src/domain/**"
disallow:
  - from: "src/web/**"
    to:   "src/infra/**"
  - from: "src/domain/**"
    to:   "src/web/**"
```

## Niche it occupies

**Polyglot file-level architecture linter + visualiser** — the same
binary handles JS/TS/Python/Rust import graphs and enforces
directory-to-directory edge rules. Closest neighbours in this
catalog:

- [`grep-ast`](../grep-ast/) / [`ast-grep`](../ast-grep/) /
  [`comby`](../comby/) — operate on syntax trees inside files.
  dep-tree operates on the *graph between files*; they answer
  "show me every call to `foo`", dep-tree answers "show me every
  edge from layer A to layer B".
- [`tokei`](../tokei/) / [`scc`](../scc/) / [`cloc`](../cloc/) /
  [`onefetch`](../onefetch/) — count lines / languages / commits.
  Orthogonal: those summarise *volume*, dep-tree summarises
  *structure*.
- [`knip`](../knip/) — dead-code / unused-export finder for
  JS/TS. Complementary: knip removes nodes from the graph (unused
  files / exports), dep-tree polices the edges that remain.
- [`madge`](https://github.com/pahen/madge) (not in this catalog) —
  the JS/TS-only ancestor of this niche. dep-tree is the
  multi-language, lint-mode-included successor.
- [`tach`](../tach/) — Python-only architecture boundary enforcer
  with a similar config-driven `check` UX. Pick `tach` for a
  Python-only repo that wants the most idiomatic Python tooling
  (it understands `__init__.py` re-exports natively); pick
  `dep-tree` when the same repo also has a TS frontend or Rust
  service and you want one binary policing both.

Pairs cleanly with [`reviewdog`](../reviewdog/) (turn
`dep-tree check` failures into PR review comments) and with
[`lefthook`](../lefthook/) /
[`pre-commit`](../pre-commit/) (run `dep-tree check` on staged
files before they reach CI).

Caveats: `render` mode is a developer-machine luxury, not something
you run in CI; the graph is *static* (no runtime / dynamic-import
resolution — `require(name)` where `name` is computed will be
missed); on monorepos with thousands of files the initial parse
takes a few seconds and consumes proportional RAM; the lint config
is glob-based, so very fine-grained "this one file only" rules end
up verbose.

## Citation

- Repo: <https://github.com/gabotechs/dep-tree>
- Latest release: **v0.23.4**
- License: **MIT**
- License file: [LICENSE](https://github.com/gabotechs/dep-tree/blob/main/LICENSE)
