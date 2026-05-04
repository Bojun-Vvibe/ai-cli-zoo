# gitlint

## Overview

`gitlint` is a commit-message linter written in Python. It enforces a
set of rules against the *message* of a commit (title length, body
wrapping, subject case, imperative mood, trailing period, allowed and
disallowed words, custom regex rules) so a repo's commit log stays
consistent without depending on every contributor reading and
remembering the contribution guide.

It runs in three modes that cover the realistic touchpoints:

- `gitlint --staged --msg-filename .git/COMMIT_EDITMSG` from a
  `commit-msg` git hook, so a malformed message is rejected at
  `git commit` time and rewritten in `$EDITOR` before any push;
- `gitlint --commits HEAD~10..HEAD` in CI, gating an entire range of
  commits on a PR before merge;
- `gitlint --hooks install` to drop a managed hook into the local
  `.git/hooks/commit-msg` for repos that haven't standardised on
  [`pre-commit`](../pre-commit/).

The default rule set is the conservative shape of "subject ≤ 72
chars, body wraps at 80, blank line between subject and body, no
trailing period on subject, no leading whitespace, no `WIP`/`fixup`
in the subject of a non-fixup commit". A `.gitlint` file at the repo
root tunes individual rules, disables any of them, or registers
custom rules as Python entry points (`user_rules` directory of small
classes implementing `validate(message) -> [RuleViolation]`).

## Repo URL

https://github.com/jorisroovers/gitlint

## Version

v0.19.1 (released 2023-03-10)

## License

MIT — upstream LICENSE file:
[`LICENSE`](https://github.com/jorisroovers/gitlint/blob/main/LICENSE).

## Install

pipx (recommended — keeps `gitlint` isolated from project Pythons):

```bash
pipx install gitlint
```

pip (into the current env):

```bash
pip install --user gitlint
```

Homebrew:

```bash
brew install gitlint
```

Verify:

```bash
gitlint --version    # gitlint, version 0.19.1
```

Generate a starter config:

```bash
gitlint generate-config > .gitlint
```

Wire it into a `commit-msg` hook (project-local, no external runner):

```bash
gitlint install-hook
```

Or run from CI against a PR's commit range:

```bash
gitlint --commits "${BASE_SHA}..${HEAD_SHA}" --fail-without-commits
```

A typical `.gitlint` to enforce
[Conventional Commits](https://www.conventionalcommits.org)-shaped
subjects:

```ini
[general]
ignore = body-is-missing
contrib = contrib-title-conventional-commits

[contrib-title-conventional-commits]
types = feat,fix,chore,docs,refactor,test,perf,build,ci,revert

[title-max-length]
line-length = 72
```

## Why use it

Three things `gitlint` does that beat the "we'll just remind people in
code review" alternative:

1. **Pre-merge enforcement at the commit-message layer.** Most lint
   tools target source code; `gitlint` is the rare tool that targets
   the commit log itself. A repo whose commit log is consistent for
   five years is a repo where `git log --oneline`, `git blame`,
   release-note generators, and changelog tools all keep working
   without curation. Adopting `gitlint` *now* costs one PR; cleaning
   up an inconsistent ten-year log later is a multi-week project.
2. **Conventional Commits without a heavy framework.** The bundled
   `contrib-title-conventional-commits` rule enforces the
   `<type>(<scope>): <subject>` shape that downstream tools
   ([`commitizen`](../commitizen/), `semantic-release`,
   `release-please`) rely on, without committing to a specific
   release-automation pipeline. So the repo gains the option of
   automated changelogs / SemVer bumps later without rewriting
   history.
3. **User rules as plain Python.** A custom rule (e.g. "every commit
   subject must reference an issue number that exists in our tracker")
   is a 20-line Python class registered in a `user_rules/` directory.
   No DSL, no plugin framework — just a class with a `validate`
   method. Easier to maintain than a `commit-msg` hook hand-written
   in `bash` + `grep` + `curl`.

## Vs Already Cataloged

- **Vs [`commitizen`](../commitizen/):** complementary, not
  competing. `commitizen` is the *interactive prompt* that
  walks a contributor through composing a Conventional-Commits-shaped
  message (`cz commit` opens a wizard); `gitlint` is the *gate* that
  rejects messages that don't conform regardless of how they were
  written. Pair them: `commitizen` for the carrot (easy to do the
  right thing), `gitlint` for the stick (impossible to merge the
  wrong thing).
- **Vs [`pre-commit`](../pre-commit/):** orthogonal — `pre-commit` is
  a hook *runner* (manages `pre-commit`, `pre-push`, `commit-msg`
  hooks across many tools); `gitlint` is one of the things you'd run
  *as* a `commit-msg` hook. The canonical wiring is to add `gitlint`
  to `.pre-commit-config.yaml` under the `commit-msg` stage.
- **Vs [`gitleaks`](../gitleaks/) / [`trufflehog`](../trufflehog/):**
  orthogonal axes — those scan commit *content* for secrets,
  `gitlint` lints commit *messages* for shape. A serious repo runs
  all three: `gitleaks` (cheap regex on every commit), `trufflehog
  --only-verified` (nightly, broader detector set), `gitlint`
  (`commit-msg` hook + CI gate on PR).

## Caveats

- **Last release was 2023-03-10 (v0.19.1).** The project still
  receives occasional fixes on `main` but cadence is slow. The
  feature surface is stable and the rule set covers the standard
  Conventional Commits / Linux-kernel-style commit message
  conventions; if you need a rule that doesn't exist, the
  `user_rules` extension point is the supported path.
- **Python-only runtime.** Unlike Go-binary peers
  ([`commitlint-rs`](https://github.com/keisku/commitlint-rs),
  [`gommit`](https://github.com/antham/gommit)), `gitlint` needs
  Python ≥ 3.7 on every machine that runs the hook. For a polyglot
  team where some contributors don't have Python set up, install via
  `pipx` or run inside `pre-commit` (which manages its own Python
  env).
- **Default rule set is opinionated.** First-time adopters on an
  existing repo should start with `gitlint --commits HEAD~50..HEAD`
  to see the full violation set, then either fix or `[general]
  ignore = ...` the rules that don't match the team's convention.
  Don't enable the `commit-msg` hook before the team has agreed on
  the active rule set, or every contributor's first commit after
  installation will be rejected.
