# yamllint

> **A linter for YAML files focused on syntax validity and stylistic
> consistency.** Single Python package providing the `yamllint` CLI: parses
> any `.yml` / `.yaml` file with the same `PyYAML`-derived parser used by
> Ansible / SaltStack / Kubernetes manifests / GitHub Actions workflows /
> docker-compose, and reports both *parse errors* (truly invalid YAML) and
> *style violations* (inconsistent indent, trailing spaces, line length,
> truthy ambiguity, document-start marker, key duplication, comment
> formatting). Pinned to **v1.38.0** (released 2026-01-13, SPDX:
> `GPL-3.0-or-later`,
> [LICENSE](https://github.com/adrienverge/yamllint/blob/master/LICENSE)).

Source: <https://github.com/adrienverge/yamllint>

## Repo

- URL: <https://github.com/adrienverge/yamllint>
- Owner: adrienverge (Adrien Vergé, individual maintainer; long-stable
  governance, ~13 years of releases)
- License file:
  [LICENSE](https://github.com/adrienverge/yamllint/blob/master/LICENSE)

## Version

`v1.38.0` — released 2026-01-13. Verify with `yamllint --version`. Stable
1.x line, semver-respecting; rule additions land as opt-in entries in
`extends: default` so upgrades almost never break a green CI.

## License

**GPL-3.0-or-later** — copyleft. Running it (CI, pre-commit, editor) imposes
no obligation; *redistributing a modified version* triggers the GPL share-
alike clause. For pure tool use this is a non-issue. Vendoring the source
into a closed-source product is constrained — invoke the published binary /
package instead.

## What it does

`yamllint file.yaml` walks the document, applies a configurable rule set,
and prints `path:line:col: [level] message (rule-id)` lines (parsable by
any editor's quickfix / by [`reviewdog`](../reviewdog/) `-f=yamllint`).
Default rule pack covers: `anchors`, `braces`, `brackets`, `colons`,
`commas`, `comments`, `comments-indentation`, `document-end`,
`document-start`, `empty-lines`, `empty-values`, `float-values`,
`hyphens`, `indentation`, `key-duplicates`, `key-ordering`, `line-length`,
`new-line-at-end-of-file`, `new-lines`, `octal-values`, `quoted-strings`,
`trailing-spaces`, `truthy`. Each rule has tunable parameters; the project
config (`.yamllint` / `.yamllint.yaml` / `.yamllint.yml` at repo root) can
extend `default` / `relaxed` and override per-rule severity (`error` /
`warning` / `disable`).

The big win versus "just run the YAML parser" is the **`truthy` rule** —
catches the unquoted `no` / `yes` / `on` / `off` / `y` / `n` that YAML 1.1
silently coerces to booleans (the famous Norway problem: country code `NO`
parsed as `false`), one of the most common production-incident classes for
Ansible / Helm / GitHub Actions YAML. The **`key-duplicates`** rule catches
the silent override where two same-named keys in a mapping leave only the
last value standing — the parser accepts it, your config silently differs
from what you read.

## When to use

- **You ship Ansible playbooks, Helm values, Kubernetes manifests, GitHub
  Actions workflows, or docker-compose files** and want a deterministic CI
  gate against the YAML-1.1 truthy footguns + duplicate-key class of bugs.
- **The team disagrees about indentation / quoting style** and you want a
  checked-in `.yamllint` to end the bikeshed in code review.
- **You already use [`pre-commit`](../pre-commit/)** — the official
  `pre-commit` hook is one entry in `.pre-commit-config.yaml` and runs in
  ~50ms on a typical repo.
- **Pair with [`reviewdog`](../reviewdog/)** to post yamllint findings as
  inline PR comments with `-filter-mode=added` so legacy violations don't
  block a PR that didn't introduce them.

## When NOT to use

- **Schema validation is the actual need.** yamllint validates *YAML
  syntax + style*, not "this is a valid Kubernetes Deployment manifest" or
  "this is a valid GitHub Actions workflow". For schema use
  [`kubeconform`](../kubeconform/) (Kubernetes), [`actionlint`](../actionlint/)
  (GitHub Actions), `helm lint` (Helm), `ansible-lint` (Ansible).
- **You only ship one or two YAML files** and the contents are stable.
  yamllint pays back at scale, not at file count one.
- **You want auto-formatting**, not just reporting. yamllint reports;
  [`yamlfmt`](../yamlfmt/) (Google) or [`prettier`](https://prettier.io/)
  rewrites. Run `yamlfmt` first, `yamllint` second to fail on what
  formatting cannot fix (semantic issues like `key-duplicates` and
  `truthy`).
- **GPL-3.0 in your distribution policy is a problem.** Use Google's
  `yamlfmt` (MIT) for the formatter side and ship yamllint as an external
  process invoked by CI rather than vendored.

## Alternatives in this catalog

- [`yamlfmt`](../yamlfmt/) — Google's YAML *formatter* (MIT). Orthogonal:
  yamlfmt rewrites for style, yamllint reports for correctness. Standard
  pipeline runs both — yamlfmt in pre-commit, yamllint in CI to catch
  semantic violations that no formatter can fix.
- [`reviewdog`](../reviewdog/) — universal lint-output → PR-comment poster
  with native `-f=yamllint` parser. Wire as
  `yamllint -f parsable . | reviewdog -f=yamllint -reporter=github-pr-review`.
- [`actionlint`](../actionlint/) — schema + expression-language linter for
  GitHub Actions workflows specifically. Run *both*: yamllint for the YAML
  layer, actionlint for the GH Actions semantics on top.
- [`kubeconform`](../kubeconform/) — Kubernetes manifest schema validation.
  Same complementary story: yamllint catches `truthy: yes` (silent boolean),
  kubeconform catches `kind: Deploymnt` (typo against the OpenAPI schema).
- [`pre-commit`](../pre-commit/) / [`lefthook`](../lefthook/) — hook
  orchestrators that drive yamllint on changed files locally before CI sees
  them.
- [`taplo`](../taplo/) for TOML, [`sqruff`](../sqruff/) for SQL,
  [`shellcheck`](../shellcheck/) for shell — the "structured-config linter
  per language" family yamllint sits in.
