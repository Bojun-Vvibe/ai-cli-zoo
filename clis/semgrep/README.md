# semgrep

> **Polyglot pattern-based static analyser whose rule language *is*
> the target language: write `$X == $X` and it matches every
> tautological self-comparison in Python, Go, Java, JS/TS, Ruby,
> C/C++, Rust, PHP, Scala, Kotlin, Swift, Terraform, Bash, YAML,
> Dockerfile, and ~30 more — same pattern, every language, no
> regex.** Rule packs cover OWASP Top 10, secrets, supply-chain
> misconfigs, and per-framework taint flows (Django, Flask, Express,
> Rails, Spring, Gin, FastAPI, …). Pinned to **v1.161.0** (released
> 2026-04-22, LGPL-2.1 —
> [LICENSE](https://github.com/semgrep/semgrep/blob/v1.161.0/LICENSE)).

Source: <https://github.com/semgrep/semgrep>

## TL;DR

`semgrep --config auto` walks your repo, picks rule packs based on
detected languages and frameworks, and prints a list of findings
with file path + line + a one-paragraph explanation of *why* the
match matters. The pattern engine matches on a real syntax tree
(tree-sitter for most langs, OCaml-native parsers for the core
ones), so `$X == $X` matches `count == count` regardless of what
identifier the local code uses, and `requests.get($URL,
verify=False)` matches every TLS-bypass call no matter how the
arguments are formatted. Inter-procedural taint mode (`mode:
taint`) follows tainted values through assignments and function
calls within the same file.

## Install

```sh
# pipx (single static binary on most platforms)
pipx install semgrep==1.161.0

# Or Homebrew
brew install semgrep

# Or Docker (matches CI exactly, no Python install)
docker run --rm -v "$PWD:/src" returntocorp/semgrep:1.161.0 \
  semgrep --config auto

# Verify
semgrep --version    # 1.161.0
```

## License

LGPL-2.1-only for the OSS engine + community rule packs — see
[LICENSE](https://github.com/semgrep/semgrep/blob/v1.161.0/LICENSE).
The Pro engine (cross-file taint, deeper interprocedural analysis,
some language frontends) and the hosted Semgrep AppSec Platform are
commercial; the LGPL CLI is fully usable on its own and is what most
CI integrations actually run. LGPL is fine for embedding the CLI as
a CI step; consult counsel if you intend to link `semgrep` as a
library into a redistributed product.

## Primary use case

You have a polyglot repo (a Go service, a TS frontend, some Python
ETL, a Terraform stack) and you want one tool that finds the same
*class* of bug — hardcoded secret, SQL string-concat, missing CSRF
token, `subprocess.run(shell=True, …)` with user input, AWS S3
bucket with public ACL, K8s deployment with `runAsRoot: true` — in
every language, with one ruleset format, one CLI, and one output
schema. `semgrep --config auto` is the floor; `semgrep --config
p/owasp-top-ten --config p/secrets --config terraform` is a typical
production setup.

## What it competes with

- **[`ast-grep`](../ast-grep/)** — single Rust binary, faster, the
  pattern syntax is similar. The split: `ast-grep` is *syntactic*
  (find-and-fix patterns), `semgrep` adds *semantic* features —
  taint analysis, dataflow, framework-aware rules, a curated
  registry of ~2,000 maintained rules. Pick `ast-grep` for
  large-scale syntactic refactors and per-repo lint catalogs; pick
  `semgrep` when "is this user input reaching `os.system`?" is the
  question, not "rewrite all `var` to `let`."
- **[`codeql`](https://codeql.github.com)** — much deeper
  inter-procedural dataflow, single-vendor (GitHub-owned), DSL is a
  full Datalog-flavoured query language with a steep learning
  curve, slower (minutes-to-hours per scan vs. `semgrep`'s seconds).
  Pick CodeQL when you have a security team writing custom queries
  for high-assurance work; pick `semgrep` when the rule author is
  an app developer and the rule needs to ship in a PR check.
- **Per-language linters (`ruff`, `eslint`, `golangci-lint`,
  `rubocop`, `clippy`)** — better at language-idiom enforcement and
  style. Worse at security patterns and at portability across the
  repo. They complement `semgrep`; they do not replace it. Pair
  [`ruff`](../ruff/) for Python style + `semgrep` for cross-language
  security.
- **[`trivy`](../trivy/) / [`grype`](../grype/) /
  [`osv-scanner`](../osv-scanner/)** — *vulnerability* scanners on
  artifacts (containers, lockfiles). They answer "do my deps have
  CVEs"; `semgrep` answers "is *my* code calling `eval(user_input)`."
  Both belong in CI; neither replaces the other.
- **[`trufflehog`](../trufflehog/) /
  [`gitleaks`](../gitleaks/) / [`ggshield`](../ggshield/)** —
  secret scanners specifically. `semgrep` has secret-pattern rules
  too (`p/secrets`) but the dedicated tools are stronger here
  (verifier-driven, deeper detector catalogs). Pick a dedicated
  secret scanner for secrets; `semgrep` for everything else.
- **[`shellcheck`](https://www.shellcheck.net/) /
  [`hadolint`](https://github.com/hadolint/hadolint)** — superior
  inside their narrow niches (bash, Dockerfile). `semgrep` handles
  both but with shallower per-language rule depth. Run all three.

## AI-native angle

`semgrep` is one of the cleanest *deterministic* feedback channels
you can give a coding agent that is editing security-sensitive
code:

- **Rules are short, readable YAML.** A rule is a pattern + a
  message + a fix template + a severity. An agent can be handed
  `~/.semgrep/myrules/` and asked to author a new rule for a
  newly-discovered footgun in the codebase; the rule lands as a
  PR like any other.
- **`--autofix` is structured, not freeform.** Where applicable, a
  rule carries a `fix:` template that `semgrep --autofix` applies
  in place. An agent that does not trust an LLM rewrite can apply
  the deterministic autofix instead and keep the LLM for cases the
  rule author flagged as `ambiguous: true`.
- **JSON output is the agent surface.** `semgrep --json` emits
  `{check_id, path, start, end, extra: {message, lines, fix,
  metadata: {cwe, owasp, ...}}}` per finding. An agent can ingest
  the JSON, prioritise by severity + CWE, and either fix or open
  issues — no scraping needed.
- **Rule registry is the prompt library, version-controlled.** The
  `p/owasp-top-ten`, `p/secrets`, `p/django`, `p/flask`,
  `p/javascript`, `p/typescript`, `p/golang`, `p/terraform`,
  `p/dockerfile`, `p/kubernetes` packs are git repos at
  <https://github.com/semgrep/semgrep-rules>. Agents can read the
  YAML directly to learn what "good" looks like in the project's
  language and framework.
- **Pair with [`patchwork`](../patchwork/) `AutoFix` Patchflow.**
  That flow runs `semgrep` in CI, feeds findings into an LLM, and
  opens a PR per fix. The deterministic part (matching the bug
  pattern) stays with `semgrep`; the indeterministic part (writing
  a clean fix when no autofix template applies) is the LLM's job.
  Clean separation of where determinism lives vs. where judgement
  lives.
- **MCP server angle.** `semgrep` exposes itself over MCP via a
  third-party `semgrep-mcp` server, giving MCP-aware agents
  (`opencode`, `claude-code`, `crush`, `goose`) "scan this diff"
  and "explain this finding" as first-class tool calls. Cleaner
  than asking an agent to shell out.

## Caveats

- **OSS engine is single-file.** Inter-procedural taint is
  intra-file in the LGPL CLI; cross-file taint requires the Pro
  engine. Plenty of real bugs are intra-file (the `request.GET[…]
  → cursor.execute(…)` SQL-i in one Django view), so the OSS
  engine catches a lot, but if your bug class is "tainted value
  flows through five files," upgrade or use CodeQL.
- **False positives are real.** A pattern-based scanner over a
  fuzzy syntax tree will flap on macro-heavy C/C++, on TS with
  unusual generics, and on framework-specific DSLs the rule
  author did not anticipate. Triage first runs as warnings, not
  PR blockers, until the false-positive rate is known per repo.
- **Tree-sitter coverage is uneven.** Java / JS / TS / Python / Go
  / Ruby / C / Terraform are first-class; less-popular languages
  (Lua, Erlang, Elixir, Clojure) are best-effort. Run a one-off
  scan first to confirm parser quality on your repo before wiring
  a CI gate.
- **`--config auto` phones home.** It fetches rule packs from the
  Semgrep registry on every run. For air-gapped environments,
  vendor `p/*` packs into the repo and pass `--config
  ./semgrep-rules/p/owasp-top-ten`.
- **Performance scales with rule × file × pattern complexity.**
  Cold scans of a large monorepo can take minutes; use
  `--exclude` aggressively, run on diffs (`semgrep ci`), and
  cache the rule registry between runs.

## Concrete example

PR gate that scans only the diff against `main`, with autofix and
JSON for downstream tooling:

```yaml
# .github/workflows/semgrep.yml
name: semgrep
on:
  pull_request: {}
jobs:
  semgrep:
    runs-on: ubuntu-22.04
    container: returntocorp/semgrep:1.161.0
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - run: |
          semgrep ci \
            --config p/default \
            --config p/owasp-top-ten \
            --config p/secrets \
            --config p/dockerfile \
            --config p/terraform \
            --baseline-ref origin/main \
            --json --json-output=semgrep.json
      - if: always()
        uses: actions/upload-artifact@v4
        with: { name: semgrep-findings, path: semgrep.json }
```

A custom local rule (`semgrep-rules/no-shell-true.yml`) blocking
`subprocess.run(shell=True)` on dynamic input:

```yaml
rules:
  - id: no-subprocess-shell-true
    languages: [python]
    severity: ERROR
    message: |
      subprocess.run(..., shell=True) with non-literal first arg is
      a command-injection sink. Pass an argv list instead.
    pattern-either:
      - pattern: subprocess.run($X, shell=True, ...)
      - pattern: subprocess.Popen($X, shell=True, ...)
      - pattern: subprocess.call($X, shell=True, ...)
    pattern-not: subprocess.run("...", shell=True, ...)
    fix: |
      subprocess.run(shlex.split($X), ...)
    metadata:
      cwe: CWE-78
      owasp: A03:2021
```

`semgrep --config semgrep-rules/no-shell-true.yml --autofix .`
finds and rewrites the violations; the same rule files travel with
the repo so contributors and CI use identical config.
