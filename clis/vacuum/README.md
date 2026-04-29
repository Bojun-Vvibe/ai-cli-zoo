# vacuum

> **The world's fastest OpenAPI 3 / Swagger 2 linter and quality
> tool** — single Go binary that reads an OpenAPI spec and reports
> ~50 built-in style / correctness rules (Spectral-compatible
> ruleset format) in milliseconds, with a TUI dashboard, HTML
> report, JSON / SARIF output, and a `vacuum lint` exit code that
> drops cleanly into CI. Pinned to **v0.26.4** (SPDX: `MIT`,
> [LICENSE](https://github.com/daveshanley/vacuum/blob/main/LICENSE)).

Source: <https://github.com/daveshanley/vacuum>

## TL;DR

`vacuum` is what you reach for when `spectral lint` is too slow
on a 30 000-line OpenAPI spec, when you want a TUI to *browse*
rule violations interactively, or when you need an HTML report
to hand to API consumers. It loads its own ruleset by default
("recommended" + "owasp" + "all"), accepts Spectral-format
rulesets unchanged so existing teams can migrate without
rewriting rules, and bundles a `vacuum dashboard` TUI that lets
you walk every violation by category with `j`/`k`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install vacuum

# Go
go install github.com/daveshanley/vacuum@latest

# Pre-built binary (release tarball)
curl -Lo vacuum.tar.gz "https://github.com/daveshanley/vacuum/releases/download/v0.26.4/vacuum_0.26.4_darwin_arm64.tar.gz"
tar xf vacuum.tar.gz
sudo install vacuum /usr/local/bin/

# Docker
docker pull dshanley/vacuum:0.26.4

# verify
vacuum version    # 0.26.4
```

## License

MIT — see
[LICENSE](https://github.com/daveshanley/vacuum/blob/main/LICENSE).

## One Concrete Example

```bash
# 1. lint a spec with the built-in recommended ruleset
vacuum lint openapi.yaml

# 2. lint with details (show the rule, file, line, why it failed)
vacuum lint -d openapi.yaml

# 3. CI mode: exit non-zero on any error, machine-readable output
vacuum lint --fail-severity error openapi.yaml

# 4. write a self-contained HTML report you can open in a browser
vacuum html-report openapi.yaml report.html

# 5. interactive TUI to browse every violation by category
vacuum dashboard openapi.yaml

# 6. emit JSON for downstream tooling
vacuum lint -j openapi.yaml > report.json

# 7. emit SARIF for code-scanning UIs
vacuum lint -r sarif openapi.yaml > report.sarif

# 8. use a Spectral-format custom ruleset
vacuum lint -r ruleset.yaml openapi.yaml

# 9. lint only the OWASP API Security rules
vacuum lint --ruleset owasp openapi.yaml

# 10. spec stats (operations, schemas, parameter counts)
vacuum spectral-report openapi.yaml
```

## Niche It Fills

**Fast OpenAPI linting that scales to enterprise specs.**
Spectral (the de-facto standard) is Node-based and slow on large
specs (multi-second startup, multi-minute lint on a 1 MB spec).
`vacuum` is a single Go binary that lints the same spec in
milliseconds, accepts Spectral rulesets unchanged, and adds a
TUI / HTML report Spectral does not have.

## When to use

1. **You already use Spectral and the lint step is the bottleneck
   in CI.** `vacuum` is a drop-in: same ruleset format, faster
   execution, identical pass/fail semantics for the rules it
   implements.
2. **You want one binary, no Node toolchain.** Static Go binary
   ships clean to a CI runner, a Docker layer, or a developer
   laptop without `npm install`.
3. **You need a report you can hand to a non-engineer.** The
   HTML report is self-contained, navigable, and shows the
   offending YAML inline.
4. **You want to browse the spec interactively.** `vacuum
   dashboard` is the only TUI in this niche.

## When NOT to use

- **You depend on a Spectral rule `vacuum` has not implemented
  yet.** The built-in ruleset overlaps heavily with Spectral's
  but is not a 100 % superset; check `vacuum lint --list-rules`
  against your custom ruleset first.
- **AsyncAPI / GraphQL / gRPC specs.** OpenAPI 3 and Swagger 2
  only. For AsyncAPI use the AsyncAPI CLI; for GraphQL use
  GraphQL Inspector.
- **You need *generation* (SDKs, server stubs, docs).** This is
  a *lint-only* tool. Pair with `openapi-generator` or `oapi-
  codegen` for codegen.

## Vs Already Cataloged

- **Vs [`bruno`](../bruno/):** orthogonal — `bruno` is an HTTP
  client / API tester, `vacuum` is a static linter. Pair: lint
  the spec with `vacuum` in CI, then exercise endpoints with
  `bruno` in your test suite.
- **Vs [`hurl`](../hurl/) / [`k6`](../k6/):** also orthogonal —
  those are runtime / load tools. `vacuum` runs before any
  request is ever sent.
- **Vs [`sqlfluff`](../sqlfluff/) (different domain, same idea):**
  conceptually identical — declarative ruleset over a
  structured-text spec, fast static lint, CI-friendly exit codes.

## Caveats

- **YAML anchors / `$ref` cycles can confuse the resolver.**
  Most real-world specs are fine, but heavily ref-recursive
  schemas may need `--skip-check` for the affected rules.
- **Built-in `owasp` ruleset is opinionated.** It will flag
  perfectly valid public APIs that intentionally do not require
  auth on certain endpoints. Suppress per-rule with
  `--ignore-file`.
- **HTML report is single-file but heavy.** A multi-MB spec
  produces a multi-MB report. Acceptable for handoff; not for
  embedding in a dashboard tile.
- **Not a fixer.** `vacuum` reports; it does not auto-edit your
  spec. For mechanical fixes, combine with `yq` or a hand-rolled
  jsonpath patch.
