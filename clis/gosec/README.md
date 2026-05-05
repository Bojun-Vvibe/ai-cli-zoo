# gosec

> **A SAST scanner for Go source — AST-walking,
> rule-based, single-binary** — analyses Go packages
> against ~30 built-in rules covering hardcoded
> credentials, weak crypto (MD5 / SHA-1 / DES / RC4),
> SQL-injection-shaped string concatenation, unsafe
> `exec.Command` usage, world-writable file modes,
> integer-overflow conversions, TLS misconfiguration,
> and the rest of the OWASP Go Secure Coding cheatsheet,
> emitting JSON / SARIF / JUnit / CSV / HTML / text.
> Pinned to **v2.26.1** (commit
> `9c4bb74a6258fe3818db8354b376ecca985704aa`,
> [LICENSE.txt](https://github.com/securego/gosec/blob/master/LICENSE.txt),
> Apache-2.0).

Source: <https://github.com/securego/gosec>

## TL;DR

`gosec` is the security-focused linter for Go that runs
in CI alongside `go vet` and `staticcheck` to catch the
classes of mistake those tools intentionally don't flag:
secrets baked into source (`var apiKey = "sk-..."`),
crypto APIs that have been deprecated for a decade
(`crypto/md5`, `crypto/sha1`, `crypto/des`),
`fmt.Sprintf`-shaped SQL where a parameterised
`db.Query(?, ...)` would be safer, `exec.Command(userInput)`
where `exec.Command(fixed, userInput)` is correct,
`os.OpenFile(..., 0666)` on a key file, and the
~30 other patterns the project tracks as `G101` through
`G505`. The output ships in machine-readable formats
(SARIF for GitHub code scanning, JUnit for CI test
panels, JSON for custom dashboards) so a finding shows
up in the same UI a developer already monitors instead
of in a separate "security report" inbox they don't
read.

## Install

```bash
# Homebrew (macOS / Linux)
brew install gosec

# Go install
go install github.com/securego/gosec/v2/cmd/gosec@v2.26.1

# Curl-pipe-bash from the official installer
curl -sfL https://raw.githubusercontent.com/securego/gosec/master/install.sh | sh -s v2.26.1

# Docker
docker run --rm -v "$PWD:/src" -w /src securego/gosec:2.26.1 ./...

# verify
gosec --version    # Version: 2.26.1
```

No daemon, no network calls (no telemetry, no rule
fetching at runtime — rules are compiled into the
binary), no privileged operations. The Docker image is
the cleanest way to run it from a CI matrix where Go
isn't otherwise installed.

## License

Apache-2.0 — see
[LICENSE.txt](https://github.com/securego/gosec/blob/master/LICENSE.txt).
Permissive with patent grant; redistribute, embed, fork
freely with attribution.

## One Concrete Example

```bash
# 1. scan the whole module from the repo root
gosec ./...

# 2. emit SARIF for GitHub code scanning upload
gosec -fmt sarif -out gosec.sarif ./...

# 3. emit JUnit for the CI test report panel
gosec -fmt junit-xml -out gosec.xml ./...

# 4. fail CI only on HIGH-confidence + HIGH-severity findings
gosec -severity high -confidence high ./...

# 5. exclude a specific rule globally (e.g. file perms in tests)
gosec -exclude=G302,G304 ./...

# 6. exclude one finding inline with a comment annotation
#nosec G404 -- non-crypto rand is intentional in this benchmark
n := rand.Intn(100)

# 7. include only the SQL injection rule (G201, G202)
gosec -include=G201,G202 ./...

# 8. scan only generated changed files (CI fast-path)
git diff --name-only origin/main...HEAD | grep '\.go$' \
  | xargs -r gosec -fmt json -out delta.json

# 9. write a custom rule via the analyzer plugin API
go run ./customrules/myrule.go ./...
```

## Niche It Fills

**Security-focused linting that lives in `go build`'s
neighbourhood, not in a separate "security tools" silo.**
The Go-specific universe already has `go vet`
(correctness), `staticcheck` (correctness + style),
`golangci-lint` (meta-linter aggregator); none of them
default to flagging hardcoded secrets, deprecated
crypto, or shell-injection-shaped `exec.Command` calls.
`gosec` is the missing fourth member of that quartet
— same ergonomics (`./...` for the whole module, exit
non-zero on findings, machine-readable output formats),
different rule set (the security one). Embedded as a
golangci-lint linter or run standalone in CI, it catches
the class of bugs that get into the wild as CVEs in Go
projects.

## Why use it

Three things `gosec` does that `go vet` and
`staticcheck` deliberately do not:

1. **Hardcoded-credential / weak-crypto detection out
   of the box.** `G101` flags `var apiKey =
   "AKIA..."`-shaped assignments via entropy + regex
   heuristics; `G401`-`G405` flag `crypto/md5`,
   `crypto/sha1`, `crypto/des`, `crypto/rc4`,
   `math/rand` for security purposes. `go vet`
   intentionally does not opine on these (they're not
   bugs in the type-system sense); `gosec` is the linter
   that does.
2. **SARIF + GitHub code scanning native path.** `-fmt
   sarif -out gosec.sarif` produces the exact format
   that the `github/codeql-action/upload-sarif` action
   ingests, so findings show up inline on the PR
   "Files changed" tab as annotations next to the
   offending line rather than buried in a CI log. The
   loop from "developer pushes" to "developer sees the
   finding next to their code" is one CI step.
3. **`#nosec` annotations with rule-id and rationale.**
   When a finding is intentional (e.g. `math/rand` in a
   load generator, weak hash for non-security
   bookkeeping), `// #nosec G404 -- non-crypto rand
   intentional` suppresses it inline with the rule ID
   and a human-readable reason. Reviewers see *why*
   each suppression exists in the diff — far better than
   the global-exclude-and-forget pattern.

For an LLM-CLI workflow that touches Go source (`opencode`,
`aider`, `claude-code` editing a Go repo), running `gosec
-fmt json ./...` after a code change and feeding the
findings back into the model gives the agent a tight
fix-loop on a class of bugs it would otherwise commit
without noticing.

## Vs Already Cataloged

- **Vs [`golangci-lint`](../golangci-lint/):** partial
  overlap — `golangci-lint` *includes* `gosec` as one
  of its ~50 wrapped linters, behind the `gosec` key
  in `.golangci.yml`. Run `gosec` standalone when you
  want the security report on its own (separate CI
  step, separate report, different team owner) or when
  you want a newer `gosec` than the version `golangci-lint`
  currently pins. Run it inside `golangci-lint` when you
  want one config + one CI invocation for all linters.
- **Vs [`semgrep`](../semgrep/):** orthogonal axes —
  `semgrep` is multi-language (Go, Python, JS, Java,
  ...) with a community rule registry and a
  pattern-matching DSL; `gosec` is Go-only with a fixed
  built-in rule set written in Go (`go/ast`-walking
  visitors). Use `semgrep` when the codebase is
  polyglot or when you need custom security rules
  written quickly; use `gosec` when the project is
  Go-only and you want zero-config security rules with
  Go's exact AST semantics (no false positives from
  generics or build-tag-conditional code).
- **Vs [`trivy`](../trivy/) / [`grype`](../grype/) /
  [`syft`](../syft/) /
  [`osv-scanner`](../osv-scanner/):** orthogonal — those
  scan **dependencies and container images** for
  known-vulnerable versions; `gosec` scans **your
  source code** for security-anti-pattern usage. A
  project needs both: `osv-scanner` to catch "you're
  using a vulnerable version of `golang.org/x/net`"
  and `gosec` to catch "your own code calls `exec.Command`
  with a tainted string".
- **Vs [`gitleaks`](../gitleaks/) /
  [`trufflehog`](../trufflehog/) /
  [`noseyparker`](../noseyparker/) /
  [`ripsecrets`](../ripsecrets/) /
  [`talisman`](../talisman/) /
  [`ggshield`](../ggshield/):** partial overlap on the
  hardcoded-credentials axis — those tools scan git
  history / staged changes / arbitrary files for
  secrets across all languages; `gosec` only flags
  secrets-shaped *constants in Go source*. They
  compose: secret-scanners catch the `.env` you
  accidentally committed; `gosec` catches the API key
  you typed into `var apiKey = "..."`.

## Caveats

- **Go-only.** No language fallback, no multi-language
  rules. A polyglot repo with Go + Python + TS needs
  one scanner per language (or `semgrep` to cover all
  three).
- **AST-level, not data-flow.** `gosec` flags `exec.Command(x)`
  whenever `x` is not a string literal — it does not
  trace whether `x` is reachable from user input, so
  expect false positives on internal command builders
  whose input is fully controlled. Suppress with
  `#nosec G204 -- input is hardcoded enum`.
- **`G101` (hardcoded credentials) uses entropy
  heuristics.** Long random-looking strings in test
  fixtures or vendored data files trigger it; tune
  with the `-exclude G101` flag scoped to test
  packages (`-exclude-dir testdata`) or use a custom
  config to lift the entropy threshold.
- **Rules update with releases, not at runtime.** The
  rule set is compiled into the binary; pin a version
  in CI and bump it deliberately. There is no "rule
  registry" to fetch (which is also the security
  benefit — supply-chain surface is `go install` only,
  no runtime rule download).
- **Exit code is binary (0 / non-zero).** Findings of
  any severity above the threshold cause exit 1; CI
  policy ("fail only on HIGH") has to be expressed via
  the `-severity` and `-confidence` flags before the
  scan, not by post-filtering the output.
