# reviewdog

> **Universal lint-output → code-review-comment poster.**
> A single Go binary that reads any linter's output (errorformat,
> RDJSON, RDJSONL, checkstyle, SARIF, diagnostic-format JSON, raw
> stdout) and posts the findings as inline review comments on a pull
> request — GitHub, GitHub Enterprise, GitLab MR, Gerrit, Bitbucket
> Cloud/Server, and a generic "local diff" mode for pre-commit use.
> Pinned to **v0.21.0** (SPDX: `MIT`,
> [LICENSE](https://github.com/reviewdog/reviewdog/blob/master/LICENSE)).

Source: <https://github.com/reviewdog/reviewdog>

## TL;DR

`reviewdog` answers one question — "I have N linters in CI; how do I
get their findings to show up as inline PR comments **only on the
lines this PR actually changed**, without writing N glue scripts?" —
with a single binary and a one-liner per linter:

```sh
golangci-lint run | reviewdog -f=golangci-lint -reporter=github-pr-review
eslint -f=checkstyle . | reviewdog -f=checkstyle -reporter=github-pr-check
hadolint Dockerfile | reviewdog -f=hadolint -reporter=github-pr-review
```

Three reporter modes cover the spectrum:

- `github-pr-review` / `gitlab-mr-discussion` — posts inline comments
  attached to changed lines; blocks merge via failure exit code.
- `github-pr-check` / `gitlab-mr-check` — posts via the Checks API
  (no comment noise, all findings collapsed into a single check run).
- `local` — diff-aware filter for pre-commit / pre-push: prints only
  findings on lines you just modified, no PR required.

## Install

```sh
# macOS via Homebrew
brew install reviewdog/tap/reviewdog

# Any platform via the install script (pinned)
curl -sfL https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh \
  | sh -s -- -b $HOME/.local/bin v0.21.0

# Or via Go
go install github.com/reviewdog/reviewdog/cmd/reviewdog@v0.21.0
```

Verify:

```sh
reviewdog -version
# 0.21.0
```

## License

MIT — unrestricted. Safe to embed in CI runner images, ship in
proprietary developer toolkits, vendor into a monorepo's `tools/`
directory, or distribute as part of a paid SaaS offering.

## Concrete example: 5 linters, one PR, only-changed-lines noise

A monorepo with Go, TypeScript, Dockerfiles, shell scripts, and YAML
typically ends up with 5 separate "lint failed" CI jobs that the
author has to scroll through. With `reviewdog` and a single
`.github/workflows/reviewdog.yml`:

```yaml
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      checks: write
    steps:
      - uses: actions/checkout@v4
      - run: |
          export REVIEWDOG_GITHUB_API_TOKEN=${{ secrets.GITHUB_TOKEN }}
          golangci-lint run --out-format=line-number ./... \
            | reviewdog -f=golangci-lint -reporter=github-pr-review -filter-mode=added -fail-on-error
          npx eslint -f=checkstyle . \
            | reviewdog -f=checkstyle -name=eslint -reporter=github-pr-review -filter-mode=added
          hadolint Dockerfile \
            | reviewdog -f=hadolint -reporter=github-pr-review -filter-mode=added
          shellcheck -f=checkstyle scripts/*.sh \
            | reviewdog -f=checkstyle -name=shellcheck -reporter=github-pr-review -filter-mode=added
          yamllint -f=parsable . \
            | reviewdog -efm='%f:%l:%c: [%t%*[a-z]] %m' -name=yamllint -reporter=github-pr-review -filter-mode=added
```

Result: every PR gets at most a handful of inline comments, each
attached to the line the author actually changed. `-filter-mode=added`
suppresses warnings on lines the PR did not touch (the dominant
source of PR-comment fatigue when you turn linting on in an existing
codebase). `-fail-on-error` makes the job red when any new finding
appears, so the merge button reflects reality.

## Niche

`reviewdog` covers the "PR-comment delivery" gap that sits between
three other layers, none of which solve it cleanly:

- **The linter itself** (`golangci-lint`, `eslint`, `hadolint`,
  `shellcheck`) — knows how to find issues but not how to post them
  to a PR, deduplicate against a base branch, or filter to changed
  lines.
- **Pre-commit hooks** ([`pre-commit`](../pre-commit/),
  [`lefthook`](../lefthook/)) — block the commit locally, but do
  nothing for contributors who skip the hook or for CI-only checks.
- **Bespoke CI scripts** that `awk` linter output and `curl` the
  GitHub API — every team writes their own version, every version
  has subtly different behavior on multi-line findings, range
  diagnostics, and Unicode filenames.

`reviewdog` is the focused glue: parse N linter formats, talk M PR
APIs, filter to the diff, exit non-zero on findings.

## Why use it

1. **Diff-aware noise filtering.** `-filter-mode=added` (or
   `diff_context`) means turning on a new lint rule on a 200-kloc
   codebase produces zero PR comments until someone actually touches
   a non-conforming line. This is the only realistic path to
   incrementally adopting strict linting on legacy code.
2. **One reporter abstraction across forges.** The same shell line
   works on GitHub, GitHub Enterprise, GitLab self-hosted, Gerrit,
   Bitbucket Cloud, and Bitbucket Server by changing one
   `-reporter=` flag. No per-forge glue code.
3. **30+ linter formats out of the box.** `golangci-lint`, `eslint`,
   `rubocop`, `pylint`, `flake8`, `mypy`, `pyright`, `clippy`,
   `tflint`, `hadolint`, `shellcheck`, `yamllint`, `markdownlint`,
   `stylelint`, `cppcheck`, `clang-tidy`, `detekt`, `ktlint`,
   `swiftlint`, `phpstan`, `psalm`, `ansible-lint`, plus generic
   `checkstyle`, `SARIF`, `RDJSON`, and a custom `errorformat`
   string for anything else.
4. **Local mode for pre-push.** `reviewdog -reporter=local
   -diff="git diff origin/main"` runs the same filtering offline,
   so contributors see exactly what CI will flag before they push.
5. **GitHub Action wrappers exist for every common linter.**
   `reviewdog/action-golangci-lint`,
   `reviewdog/action-eslint`,
   `reviewdog/action-shellcheck`, etc. — drop-in jobs that Just
   Work, useful as a starting point even if you eventually inline
   the binary call.

## Vs already cataloged

- **vs [`pre-commit`](../pre-commit/) / [`lefthook`](../lefthook/)**
  — those are local hook managers that run linters before the commit
  lands; `reviewdog` runs in CI (or local diff mode) and posts
  results back to the PR. Complementary: pre-commit catches it on
  the developer's laptop, reviewdog catches it for the contributors
  who skipped the hook.
- **vs [`golangci-lint`](../golangci-lint/) /
  [`shellcheck`](../shellcheck/) / [`hadolint`](../hadolint/) /
  [`actionlint`](../actionlint/) / [`yamlfmt`](../yamlfmt/)** —
  those are the linters; `reviewdog` is the delivery layer that gets
  their findings in front of a reviewer at the right granularity.
- **vs [`pr-agent`](../pr-agent/) / LLM-based PR review tools** —
  those generate prose review comments via an LLM; `reviewdog` posts
  *deterministic* linter findings. Use both: LLM for design review,
  reviewdog for "you forgot the trailing newline on line 47."
- **vs Sonar / CodeClimate / Codacy** — those are hosted SaaS code-
  quality platforms with their own dashboards and pricing tiers;
  `reviewdog` is a 6 MB binary that lives in your existing CI and
  posts directly to your existing PR. No vendor account, no second
  dashboard.

## Caveats

- **Token scopes matter.** `github-pr-review` needs `pull-requests:
  write`; `github-pr-check` needs `checks: write`. With
  `GITHUB_TOKEN` from `pull_request` events on forked PRs, GitHub
  withholds write scopes by default — use `pull_request_target` (and
  understand the security implications) or switch to the `-check`
  reporter which works with the default token.
- **Inline-comment placement requires correct line numbers.** If a
  linter emits 1-based vs 0-based lines, or attaches a finding to a
  range that crosses the diff boundary, the comment may end up on
  the "wrong" line or be silently dropped. Test new linters with
  `-reporter=local` first.
- **Per-PR comment deduplication is best-effort.** Re-running the
  workflow on the same SHA will not double-post comments, but force-
  pushes that change line numbers can leave orphan comments on the
  old commit. The `-tee` flag also dumps to stdout so the raw
  findings remain visible in the CI log.
- **Not a full code-review platform.** `reviewdog` posts findings,
  but it does not enforce required-reviewer rules, manage approvals,
  or track unresolved-comment counts. Pair with the forge's native
  branch-protection rules.
- **Last release v0.21.0 is September 2025.** Repository remains
  active on master with regular merges; the slow release cadence
  reflects API stability rather than abandonment. Pin to v0.21.0
  and re-evaluate annually.

## How `reviewdog` fits the LLM-CLI workflow

- **Agent-generated PRs:** when an LLM agent opens a PR, run the
  full linter suite through `reviewdog -reporter=github-pr-review`
  so the agent's next iteration (or a human reviewer) sees the
  exact line-by-line findings as inline comments rather than as a
  single failed-CI summary. The agent can then read the comments
  back via the GitHub API and produce a targeted fix-up commit.
- **Pre-push gate:** `reviewdog -reporter=local -diff="git diff
  origin/main"` in a pre-push hook gives the agent the same
  feedback it would get from CI, but offline and instant — useful
  when the agent is iterating in a worktree before pushing.
- **Eval scaffolding:** when measuring an agent's "did the patch
  pass lint" rate, parsing reviewdog's exit code (0 = clean, 1 =
  findings on changed lines) is more reliable than parsing N
  different linter exit codes.
- **Multi-linter consolidation:** an agent that previously had to
  understand `golangci-lint --out-format=line-number`,
  `eslint -f=checkstyle`, and `hadolint`'s native output can now
  treat all three as "run linter, pipe to reviewdog, read JSON
  diagnostics back" — one parsing path instead of three.
