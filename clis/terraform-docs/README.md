# terraform-docs

- **Repo:** https://github.com/terraform-docs/terraform-docs
- **Version:** v0.22.0 (latest stable, April 2026)
- **License:** MIT ([LICENSE](https://github.com/terraform-docs/terraform-docs/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install terraform-docs` · `go install github.com/terraform-docs/terraform-docs@v0.22.0` · `docker pull quay.io/terraform-docs/terraform-docs:0.22.0` · static binaries on the GitHub release page · binary name is `terraform-docs`

## What it does

`terraform-docs` reads a Terraform module directory — `.tf` files,
`variables.tf`, `outputs.tf`, `versions.tf`, `providers.tf`,
`README.md` if it has a generated section — and emits a structured
description of the module's **inputs, outputs, providers, required
versions, modules called, and resources managed**, in any of about a
dozen output formats: Markdown table, Markdown document, AsciiDoc
table, AsciiDoc document, JSON, YAML, XML, TOML, pretty terminal,
or a custom Go template (`--output-template`). The flagship use case
is to keep the module's `README.md` in sync with the actual
variable / output definitions: you put a pair of HTML comment markers
in the README, run `terraform-docs markdown table --output-file
README.md --output-mode inject .`, and the tool replaces just the
section between the markers with a freshly-generated table — the
rest of your hand-written README stays untouched.

It runs in three idiomatic ways: as a one-shot CLI on your laptop,
as a [`pre-commit`](https://pre-commit.com/) hook so the README can
never drift from the code (the upstream repo ships an official
`.pre-commit-hooks.yaml`), and as a CI step that fails the build if
the generated section would change. v0.22.0 added closed-ATX heading
support (`### Foo ###` instead of `### Foo`) which matters for
strict Markdown linters.

## When to pick it / when not to

Pick `terraform-docs` whenever you have **shared Terraform modules**
— internal module registries, public modules on the registry, monorepo
modules consumed by multiple stacks. The single highest-value behavior
is "the input/output table in the README is always correct, by
construction, because CI rejects PRs where it would have changed."
Module consumers learn to trust the README, which is a noticeable
quality-of-life win in any team larger than two people.

Pick it for a **module documentation pipeline** where you generate
JSON / YAML output and feed it into a static-site generator
(`terraform-docs json --output-file docs.json .` then a Hugo or
MkDocs build), or where you maintain a custom Markdown style via
`--output-template`.

Skip it when:

- You have **root modules only**, no shared modules — the value of
  auto-generated input tables drops sharply when nothing imports
  your module. A hand-written README is fine.
- You need **provider documentation** (the `provider "aws"` schema
  itself, not your usage of it) — that's
  [`terraform providers schema`](https://developer.hashicorp.com/terraform/cli/commands/providers/schema)
  + [`tfschema`](https://github.com/minamijoyo/tfschema), not
  `terraform-docs`.
- You want **policy / cost / security checks** instead of
  documentation — that's [`tflint`](https://github.com/terraform-linters/tflint),
  [`tfsec`](https://github.com/aquasecurity/tfsec) /
  [`trivy config`](https://trivy.dev/), or
  [`infracost`](https://www.infracost.io/), not this tool.
- You want **diagrams** of resources / dependencies —
  [`inframap`](https://github.com/cycloidio/inframap) or
  [`rover`](https://github.com/im2nguyen/rover), not `terraform-docs`.

## Why it matters in an AI-native workflow

LLM agents asked to "use this Terraform module" or "wire up
infrastructure for X" are catastrophically bad at it when the
README is stale or absent. They'll invent variable names, hallucinate
defaults, or — worst case — write the right-shaped block but with
the wrong `type` constraint, which Terraform happily accepts at
plan time and explodes on at apply time. `terraform-docs` solves
that at the source: a README generated from `variables.tf` cannot
drift, so the README the agent reads as context is identical to the
code the agent will be calling. JSON output is also useful as a
direct context-injection format — `terraform-docs json .` produces a
strictly-typed, schema-style document the agent can ingest without
any Markdown parsing.

A more subtle benefit: when the agent is **writing** a new module,
running `terraform-docs --output-mode inject README.md .` after each
edit gives it instant feedback on whether its variable / output
declarations are well-formed (parse failures are immediate) and
keeps the PR diff coherent (no manual table edits in the README).

## Example invocations

```bash
# Print a Markdown table of the module's inputs, outputs, providers, etc.
terraform-docs markdown table .

# Markdown document form (one section per variable, with descriptions).
terraform-docs markdown document ./modules/network

# Inject the generated section between BEGIN_TF_DOCS / END_TF_DOCS
# comment markers in the existing README.md (idempotent).
terraform-docs markdown table --output-file README.md --output-mode inject .

# Strict mode: nonzero exit if the README would change.
# Drop this in CI as a "docs are up to date" gate.
terraform-docs markdown table --output-file README.md \
    --output-mode inject --output-check .

# Machine-readable JSON for downstream tooling.
terraform-docs json . > module.json

# Pretty terminal output for ad-hoc inspection.
terraform-docs pretty .

# Use a custom Go template for non-standard Markdown styles.
terraform-docs --output-template "$(cat ./.tfdocs.tmpl)" .

# Run as a pre-commit hook (in .pre-commit-config.yaml):
#   - repo: https://github.com/terraform-docs/terraform-docs
#     rev: v0.22.0
#     hooks:
#       - id: terraform-docs-go
#         args: ["markdown", "table", "--output-file", "README.md",
#                "--output-mode", "inject", "./modules/network"]
```

## Caveats

- The injected README section is delimited by HTML comment markers
  (`<!-- BEGIN_TF_DOCS -->` / `<!-- END_TF_DOCS -->`); if you remove
  them, the next run will re-append a fresh section instead of
  replacing the existing one, which can produce duplicate tables.
  Always commit the markers.
- `--output-mode inject` requires the file to already exist; for a
  brand-new module use `--output-mode replace` once to bootstrap.
- The tool reads only `.tf` files in the directory you point it at —
  it does **not** recurse into child modules by default. If you want
  per-submodule docs, run it once per directory or use the
  `recursive` config option in `.terraform-docs.yml`.
- v0.22.0's "closed ATX headers" option (`### Foo ###`) is opt-in;
  if you rely on a Markdown linter that bans them, leave it off.
- `terraform-docs` does **not** validate that variable types or
  defaults are sensible — that is `tflint`'s job. Use both in CI;
  they don't overlap.
