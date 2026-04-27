# git-cliff

> **Conventional-commits-aware changelog generator that walks `git log`
> between two refs, parses commit messages against a configurable
> regex grammar, groups them by type (feat / fix / docs / perf / …),
> and renders a Markdown / JSON / GitHub-release-shaped changelog
> from a Tera template.** Pinned to **v2.13.1**, Apache-2.0
> ([LICENSE-APACHE](https://github.com/orhun/git-cliff/blob/main/LICENSE-APACHE)).

- **Repo:** https://github.com/orhun/git-cliff
- **Latest version:** v2.13.1
- **License:** Apache-2.0 (`LICENSE-APACHE` at repo root, dual-licensed
  with MIT via `LICENSE-MIT`; SPDX `Apache-2.0 OR MIT`)
- **Category:** `release-tooling` / `dev-experience`
- **Language:** Rust

## What it does

`git-cliff` reads the git history between two refs (default: latest
tag → `HEAD`) and produces a changelog by:

1. **Parse** — every commit subject + body matched against the
   `commit_parsers` regex list in `cliff.toml`; commits that fit the
   Conventional Commits shape (`feat:`, `fix(scope):`, `feat!:`)
   land in their declared group, anything else is skipped or routed
   to "Miscellaneous" depending on config.
2. **Group** — commits bucketed by type, ordered by configurable
   precedence (breaking changes → features → fixes → docs → perf →
   refactor → tests → build → ci → chore).
3. **Render** — a Tera template (`body` + optional `header` /
   `footer`) controls the output shape; ships with presets for
   Keep-a-Changelog, GitHub releases, GitLab releases, Gitea, Bitbucket,
   and Cocogitto formats.

Modes: `git cliff` (full history), `git cliff --unreleased` (since
last tag, the "what's about to ship" preview),
`git cliff --tag v2.0.0 --output CHANGELOG.md` (cut a release section
into the file with a date stamp), `git cliff --bump` (auto-detect
next semver from breaking-change footers + commit types and emit it),
`git cliff --context` (dump the parsed JSON shape for downstream
templating), and `--include-path` / `--exclude-path` for monorepo
per-package scoping. GitHub Actions integration ships in
`orhun/git-cliff-action` for one-step "tag → render → attach to
GitHub release" pipelines.

## Why included

Hand-curated changelogs rot the moment shipping cadence picks up;
auto-generated ones from `gh release create --generate-notes` are
flat lists with no grouping or breaking-change call-outs. `git-cliff`
sits in the middle: the *commits* are still hand-written (so the
changelog quality is bounded by commit-message discipline), but the
*structuring + rendering* is one command and one TOML file checked
into the repo. Pairs naturally with [`gptcommit`](../gptcommit/) and
[`opencommit`](../opencommit/) (LLM-generated Conventional Commits at
write time, deterministic changelog at release time) and with
[`gh-dash`](../gh-dash/) (browse PRs going into the next release in
the dashboard, run `git cliff --unreleased` to draft the notes). The
`--bump` mode is the practical answer to "what semver should this
release be?" without standing up `release-please` /
`semantic-release` infrastructure.
