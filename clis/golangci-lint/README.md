# golangci-lint

> **The de-facto meta-linter for Go** — runs ~100 individual
> linters (`govet`, `staticcheck`, `errcheck`, `revive`,
> `gosec`, `gocritic`, `unused`, `ineffassign`, `gofmt`, …)
> in one parallelized pass with shared AST/SSA/type-check
> state, deduplicated diagnostics, a single `.golangci.yml`
> config, and a single CI invocation. Pinned to **v1.62.2**
> ([LICENSE](https://github.com/golangci/golangci-lint/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/golangci/golangci-lint>

## TL;DR

`golangci-lint` is the orchestrator that every Go project of
non-trivial size eventually adopts. The Go ecosystem has a long
tail of single-purpose linters (each owning one rule family):
`staticcheck` for SA/ST checks, `errcheck` for unhandled
errors, `revive` for style, `gosec` for security AST patterns,
`gocritic` for refactor suggestions, `unused` for dead code,
`ineffassign` for ineffective assignments, `gocyclo` for
complexity, `dupl` for clones, and so on. Running each binary
separately in CI is slow (each parses the package graph and
type-checks afresh), the diagnostics overlap, and the configs
fragment. `golangci-lint` loads the AST + SSA + type-check
results *once*, runs the configured linters in parallel against
that shared state, deduplicates overlapping diagnostics, applies
include/exclude rules from one `.golangci.yml`, and prints a
unified report (`tab`, `colored-tab`, `json`, `checkstyle`,
`junit-xml`, `github-actions`, `sarif`).

## Install

```bash
# Homebrew
brew install golangci-lint

# install script (pinned)
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/HEAD/install.sh \
  | sh -s -- -b $(go env GOPATH)/bin v1.62.2

# Go install (slower, builds from source — official docs discourage this for CI)
go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.62.2

# verify
golangci-lint --version    # has version 1.62.2
```

## One Concrete Example

```yaml
# .golangci.yml at repo root — pragmatic starter config
run:
  timeout: 5m
  go: "1.23"
  tests: true
  build-tags: [integration]

linters:
  disable-all: true
  enable:
    - errcheck        # unchecked errors
    - govet           # stdlib vet
    - ineffassign     # ineffective assignments
    - staticcheck     # SA/ST/QF/S checks (the big one)
    - unused          # dead code
    - gofmt           # formatting
    - goimports       # import grouping
    - revive          # configurable replacement for golint
    - gocritic        # opinionated checks
    - gosec           # security AST patterns
    - bodyclose       # http.Response.Body.Close()
    - errorlint       # %w/errors.Is/errors.As correctness
    - exhaustive      # switch over enums
    - misspell        # typos in identifiers/comments
    - prealloc        # slice prealloc hints
    - unconvert       # redundant conversions
    - unparam         # unused function params

linters-settings:
  staticcheck:
    checks: ["all"]
  govet:
    enable-all: true
    disable: [fieldalignment]   # noisy, opt-in elsewhere
  gosec:
    excludes: [G104]            # errcheck handles this better
  revive:
    rules:
      - name: var-naming
      - name: indent-error-flow
      - name: error-return
      - name: blank-imports

issues:
  exclude-rules:
    - path: _test\.go
      linters: [errcheck, gosec, dupl, gocyclo]
    - path: cmd/
      linters: [gochecknoinits]
  max-issues-per-linter: 0
  max-same-issues:       0
```

```bash
# local dev: lint changed files vs main, with caching
golangci-lint run --new-from-rev=origin/main ./...

# CI: full sweep, machine-readable for PR annotations
golangci-lint run --out-format=github-actions,colored-tab ./...

# run only one linter (debugging a flake)
golangci-lint run --disable-all --enable=staticcheck ./...

# auto-fix the linters that support it (gofmt/goimports/misspell/unconvert/...)
golangci-lint run --fix ./...
```

The `--new-from-rev` flag is the single most important
"adoption" feature: on a brownfield codebase with thousands of
existing warnings, you turn on every linter you want, then gate
CI on `--new-from-rev=$(git merge-base origin/main HEAD)` so the
PR only fails on issues *introduced by this PR*. Existing debt
stays visible in `golangci-lint run` (no flag) but doesn't block
merges, and the codebase improves monotonically.

## Niche It Fills

**The single CI step that runs every Go static-analysis tool
worth running.** For Go repos, "we have 11 separate linters in
CI" is a smell — `golangci-lint` collapses them into one
invocation, one config, one set of suppressions, one cache, and
one PR-annotation surface. Adopted by the Kubernetes ecosystem,
HashiCorp, Grafana, CockroachDB, the standard library tooling,
and most healthy Go projects.

## Vs Already Cataloged

- **Vs [`staticcheck`](https://staticcheck.dev/) standalone:**
  Staticcheck is one of the linters golangci-lint runs (and
  arguably the highest-signal one). For a tiny project where you
  only want SA/ST checks, `staticcheck ./...` is fine. The moment
  you want a second linter alongside it, switch to
  golangci-lint to avoid double-parsing the package graph.
- **Vs [`semgrep`](https://semgrep.dev/) / `gosec` standalone /
  `codeql`:** semgrep and codeql are language-agnostic
  pattern/dataflow tools — they shine across polyglot repos and
  for security-focused custom rules. golangci-lint is Go-native
  and ships dozens of Go-aware checks (SSA-based unused detection,
  type-aware errcheck) that pattern-matchers can't easily express.
  Run both: golangci-lint for daily Go hygiene, semgrep/codeql
  for org-wide security policy.
- **Vs [`ruff`](../ruff/) / [`biome`](../biome/) / [`oxc`](../oxc/):**
  same *category* (multi-rule linter consolidating an ecosystem)
  for Python/JS-TS respectively. golangci-lint is the Go
  equivalent. If you're polyglot, you'll end up running all of
  these in CI, one per language.
- **Vs [`shellcheck`](../shellcheck/) / [`hadolint`](../hadolint/) /
  [`yamlfmt`](../yamlfmt/):** complementary, not competitive — Go
  repos almost always have shell scripts, Dockerfiles, and YAML
  that those tools handle. golangci-lint owns only `.go`.
- **Vs [`treefmt`](../treefmt/) + a fleet of formatters:** treefmt
  is a *formatter* multiplexer (orchestrates `gofmt`, `prettier`,
  `black`, `nixfmt` per file extension). golangci-lint is a
  *linter* multiplexer for one language, with much deeper
  semantic analysis than any formatter. Use both: treefmt at
  pre-commit, golangci-lint in CI.
- **Vs `go vet ./...`:** `go vet` is a small subset of what
  `govet` (the linter) reports inside golangci-lint, plus
  golangci-lint runs ~100 other things. `go vet` stays useful as
  the "no-config baseline" that ships with the toolchain; it's
  not a replacement.

## Caveats

- GPL-3.0 licensing — the *binary* you run in CI is fine
  (running a GPL tool to lint your code does not affect your
  code's license), but vendoring its packages into a closed
  product is constrained. For typical CI usage, irrelevant.
- Linter-version skew is real: each upstream linter has its own
  release cadence, and golangci-lint pins specific versions per
  release. Bumping `golangci-lint` between minor versions
  occasionally surfaces new findings (or, rarely, removes ones
  you'd suppressed). Pin the version in CI (`@v1.62.2`, not
  `@latest`) and bump deliberately.
- The default linter set (when you run `golangci-lint run` with
  no config) is *small* and conservative. Most teams enable
  more — see the example config above. The flip side: enabling
  everything (`linters: enable-all: true`) is overwhelming on
  brownfield code; pair with `--new-from-rev` or build up the
  enabled list gradually.
- The cache lives at `$XDG_CACHE_HOME/golangci-lint` (default
  `~/.cache/golangci-lint`) and can grow large on big monorepos
  (hundreds of MB). In CI, key your cache on
  `golangci-lint version + go.sum hash + .golangci.yml hash` to
  get fast warm runs without staleness.
- `--fix` only applies to the subset of linters that implement
  fixes (gofmt, goimports, misspell, unconvert, gci, godot,
  whitespace, a handful of others). Most semantic linters
  (staticcheck, gosec, errcheck) are diagnostic-only and require
  human edits.
- On generated code, exclude paths via
  `issues.exclude-rules.path` rather than littering
  `//nolint` comments — the comment form is durable but obscures
  intent and is easy to copy-paste accidentally into hand-written
  code.
- The community fork [`golangci-lint-langserver`](https://github.com/nametake/golangci-lint-langserver)
  exposes diagnostics over LSP for editor integration; out of
  scope here but worth knowing if your editor doesn't already
  bundle a golangci-lint integration.
