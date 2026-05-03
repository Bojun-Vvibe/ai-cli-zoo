# actionlint

> **Static checker for GitHub Actions workflow files — finds
> bugs in `.github/workflows/*.yml` before they fail in CI.**
> Parses the workflow YAML, validates against the Actions
> schema, type-checks expression contexts (`${{ ... }}`),
> shellchecks every `run:` block, and flags 40+ classes of
> mistakes (typoed event names, undefined `needs:` references,
> wrong matrix shapes, deprecated `set-output` syntax,
> unsupported runner labels, glob patterns that match nothing).
> Pinned to **v1.7.12**
> ([LICENSE.txt](https://github.com/rhysd/actionlint/blob/main/LICENSE.txt),
> MIT; version checked via
> `gh release view --repo rhysd/actionlint`).

Source: <https://github.com/rhysd/actionlint>

## TL;DR

`actionlint` is a single ~6 MB Go binary that reads every
workflow under `.github/workflows/`, builds an AST, and
validates it against the official Actions schema plus a
hand-curated rule set. It runs `shellcheck` on each `run:`
script (with the right shell defaulting per-runner) and
`pyflakes` on each `python` block, so a typo in a `bash` heredoc
or an undefined Python variable surfaces locally, not in the
"All checks have failed" PR badge an hour later. Output is
plain text, JSON, SARIF, or GitHub-Actions-annotation format
so it slots into PR comments and the `Files changed` tab.

## Install

```bash
# macOS / Linux via Homebrew
brew install actionlint

# Go
go install github.com/rhysd/actionlint/cmd/actionlint@v1.7.12

# Single binary (Linux x86_64)
curl -L https://github.com/rhysd/actionlint/releases/download/v1.7.12/actionlint_1.7.12_linux_amd64.tar.gz \
  | tar xz && sudo mv actionlint /usr/local/bin/

# pre-commit
# - repo: https://github.com/rhysd/actionlint
#   rev: v1.7.12
#   hooks: [{ id: actionlint }]
```

## Example

```bash
# Check every workflow in the current repo
actionlint

# Pin to one file, emit GitHub annotations
actionlint -format '{{range $err := .}}::error file={{$err.Filepath}},line={{$err.Line}}::{{$err.Message}}{{end}}' \
  .github/workflows/release.yml

# JSON for downstream tooling
actionlint -format '{{json .}}' > lint.json

# Disable shellcheck integration in air-gapped CI
actionlint -shellcheck=

# SARIF for code-scanning upload
actionlint -format '{{sarif .}}' > actionlint.sarif
```

## When to use

- You have a non-trivial `.github/workflows/` tree (more than
  one matrix, reusable workflows, composite actions, env
  expressions) and want a fast local check before `git push`.
- You've been bitten by `${{ secrets.FOO }}` typos that silently
  resolve to empty strings or `if:` expressions that always
  evaluate true.
- You want SARIF output for the Actions code-scanning surface
  without writing a custom converter.

## When NOT to use

- You only have a single trivial workflow and the cost of one
  CI round-trip is negligible.
- You're on GitLab CI / CircleCI / Jenkins / Buildkite —
  actionlint is GitHub-specific; reach for that platform's own
  validator (`gitlab-ci-lint`, `circleci config validate`,
  `bk-cli pipeline lint`) instead.
- You need policy-as-code (e.g. "no third-party action outside
  this allowlist") — pair actionlint with
  [`conftest`](../conftest/) or step-security `harden-runner`;
  actionlint validates correctness, not policy.

## Orthogonality vs existing zoo entries

- **vs [`shellcheck`](../shellcheck/)** — actionlint *embeds*
  shellcheck for `run:` blocks but adds the workflow-level
  schema, expression type-checker, and matrix/needs analysis
  that shellcheck alone cannot see.
- **vs [`yamlfmt`](../yamlfmt/)** — yamlfmt fixes whitespace
  and key ordering; actionlint cares about the semantics
  (does this event exist, does this `needs:` resolve).
- **vs [`hadolint`](../hadolint/)** — same "lint a CI artifact"
  niche, different artifact: hadolint owns Dockerfiles,
  actionlint owns workflows. Run both in `pre-commit`.
- **vs [`gh`](../gh/)** — `gh workflow view` shows runs;
  actionlint statically checks the workflow source before any
  run is queued.

## Caveats

- Schema lags upstream by days when GitHub ships a new event or
  runner label; if you hit a "unknown event" false positive,
  bump to the latest tag or pass `-ignore`.
- The `expression` type checker is conservative — some valid
  but exotic `fromJSON` patterns get flagged; suppress per-line
  with `# actionlint shellcheck disable=...` style pragmas.
- Reusable workflow inputs in another repo are validated by
  signature only; cross-repo `secrets: inherit` is trusted.
