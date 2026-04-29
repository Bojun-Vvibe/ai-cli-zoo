# markdownlint-cli2

- **Repo:** https://github.com/DavidAnson/markdownlint-cli2
- **Companion library:** https://github.com/DavidAnson/markdownlint
- **Version:** v0.22.1 (2026)
- **License:** MIT — [LICENSE](https://github.com/DavidAnson/markdownlint-cli2/blob/main/LICENSE) (SPDX: `MIT`)
- **Language:** JavaScript (Node.js)
- **Install:** `npm install -g markdownlint-cli2` · `brew install markdownlint-cli2` · `npx markdownlint-cli2 "**/*.md"` (no install) · also distributed as a Docker image, GitHub Action, and pre-commit hook · binary name is `markdownlint-cli2`

## What it does

`markdownlint-cli2` is the fast, configuration-file-driven CLI front-end to
DavidAnson's `markdownlint` library — the de-facto linter for
CommonMark / Markdown that powers VS Code's "markdownlint" extension and
ships ~50 built-in rules covering heading hierarchy (MD001 / MD003 /
MD025), list / ordered-list consistency (MD004 / MD007 / MD029 / MD030),
trailing whitespace + line length (MD009 / MD013), code-fence style
(MD040 / MD046 / MD048), broken / inconsistent links and images
(MD034 / MD039 / MD042 / MD051 / MD052 / MD053), HTML usage in Markdown
(MD033), and dozens more.

- Reads `.markdownlint-cli2.jsonc` / `.yaml` / `.cjs` / `.mjs` config so
  rule selection, glob includes / excludes, custom rules, and inline
  `markdown-it` plugins all live next to the source instead of in CI YAML.
- **`--fix`** auto-corrects the rules that have a deterministic fix
  (whitespace, blank-line counts, fence styles, list markers,
  heading-style normalisation) so the linter doubles as a formatter for
  the mechanical tail.
- **Glob-first** — `markdownlint-cli2 "**/*.md" "#node_modules"` is the
  canonical invocation, with first-class `.gitignore` honouring
  (`gitignore: true` in config) so CI runs match local dev runs.
- Integrates as a **GitHub Action**, a **pre-commit** hook, a **Docker
  image** (`davidanson/markdownlint-cli2:v0.22.1`), and the `markdownlint`
  VS Code extension uses the *same* engine, so editor diagnostics and
  CI gate cannot drift.
- Significantly faster and more configurable than the older `markdownlint-cli`
  (which is now in maintenance) — the `cli2` rewrite is the recommended
  surface and the only one that gets new features.

## Example

```sh
# Lint everything under docs/ and the root, ignore vendored content
markdownlint-cli2 "docs/**/*.md" "*.md" "!**/node_modules/**"

# Auto-fix the mechanical tail in place
markdownlint-cli2 --fix "**/*.md"

# Run with no on-disk config (one-off ad-hoc rules)
markdownlint-cli2 --config '{"MD013": false, "MD041": false}' README.md
```

## When to use it

Reach for `markdownlint-cli2` when a repo accumulates docs / READMEs /
ADRs / changelogs in Markdown and you want the same kind of CI gate
on prose that `eslint` / `ruff` / `clippy` give you on code:

- Multi-author docs repos where heading-hierarchy / list-style / link
  hygiene drifts file-by-file without enforcement.
- Any project where the README is a contract (open-source landing page,
  internal RFC template, ADR directory) and broken reference links or
  inconsistent heading levels are bugs.
- Pre-commit + CI dual-enforcement (the hook catches it locally, the
  action catches it in PRs) so contributors cannot land docs that fail
  the same rules the maintainers enforce.

Pairs with [`prettier`](https://prettier.io) (Markdown formatter for
table alignment + soft-wrap normalisation that markdownlint does not do)
and [`vale`](https://vale.sh) (prose style checker for tone /
terminology — different layer, runs against the same `.md` files
without conflict). Skip the older `markdownlint-cli` (different repo,
DavidAnson/markdownlint-cli) for new projects — `cli2` is the
maintained surface.
