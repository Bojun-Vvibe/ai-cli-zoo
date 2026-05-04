# commitizen

> **A Python CLI that turns "write a commit message" into a guided, schema-
> validated prompt — and then bumps the version + writes the changelog from
> the resulting history.** One `cz` binary covering three jobs: an
> interactive commit composer (`cz commit` walks `type` / `scope` /
> `subject` / `body` / `BREAKING CHANGE` for Conventional Commits or any
> custom rule pack), a CI lint that fails the PR when a message is off-
> spec (`cz check --rev-range origin/main..HEAD`), and a release-driver
> that reads the commits since the last tag, computes the next semver
> bump, rewrites your `pyproject.toml` / `package.json` / `Cargo.toml` /
> arbitrary `version_files`, regenerates `CHANGELOG.md`, and creates the
> tag — `cz bump --changelog` is the whole release in one command. Pinned
> to **v4.15.0** (released 2025, MIT,
> [LICENSE](https://github.com/commitizen-tools/commitizen/blob/master/LICENSE)).

Source: <https://github.com/commitizen-tools/commitizen>

## Repo

- URL: <https://github.com/commitizen-tools/commitizen>
- Owner: commitizen-tools (community-maintained Python rewrite of the
  original Node `commitizen/cz-cli`; the two projects share the
  Conventional Commits philosophy but the Python project owns the
  release-driver story end-to-end and is the one this entry covers)
- License file:
  [LICENSE](https://github.com/commitizen-tools/commitizen/blob/master/LICENSE)

## Version

`v4.15.0` — verify with `cz version`. Stable 4.x line; the rule pack
(`cz_conventional_commits` by default) is shipped in-tree, so a `pip
install --upgrade commitizen` cannot silently change what passes
`cz check`. Custom rule packs (`name = "cz_jira"`, your own subclass of
`BaseCommitizen`) live in `pyproject.toml` and survive upgrades.

## License

**MIT** — permissive. Vendor, fork, embed, ship; no copyleft obligations.
The bundled rule packs (`cz_conventional_commits`, `cz_customize`) are
also MIT.

## Install

- `pipx install commitizen` (recommended — isolates the install from your
  project's venv so `cz` is on `PATH` for any repo)
- `pip install commitizen` (project-local)
- `brew install commitizen`
- `uv tool install commitizen`
- `pre-commit` hook: one entry pointing at the
  `https://github.com/commitizen-tools/commitizen` rev — adds `cz check
  --commit-msg-file` as a `commit-msg` hook so off-spec messages fail
  *locally* before they ever land in `git log`.

## What it does

`cz commit` (alias `cz c`) replaces `git commit` with an interactive
prompt: pick a `type` (`feat` / `fix` / `docs` / `style` / `refactor` /
`perf` / `test` / `build` / `ci` / `chore` / `revert`), enter an optional
`scope`, write a one-line `subject`, optional `body`, optional
`BREAKING CHANGE:` footer, optional `Closes #123` footer — the prompt
enforces line lengths, refuses an empty subject, and writes a
Conventional-Commits-shaped message that downstream tools (changelog
generators, semver bumpers, GitHub release notes templates) can parse.

`cz check` lints existing messages: `cz check --message "feat: x"`
validates one string, `cz check --commit-msg-file .git/COMMIT_EDITMSG` is
the `commit-msg` hook shape, `cz check --rev-range origin/main..HEAD`
validates every commit on a PR branch in CI.

`cz bump` is the release driver and the reason most projects adopt
commitizen over the simpler "lint commit messages" tools:

1. Reads commits since the last tag matching `tag_format` (default `v$version`).
2. Classifies each by Conventional Commit type — any `BREAKING CHANGE` /
   `feat!` triggers a major bump, any `feat` triggers a minor bump,
   anything else is patch.
3. Rewrites `version` strings in every file listed under
   `[tool.commitizen] version_files` (works across `pyproject.toml`,
   `package.json`, `Cargo.toml`, `__init__.py`, `setup.cfg`, arbitrary
   `path/to/file.txt:version_marker`).
4. Generates / appends to `CHANGELOG.md` (Keep a Changelog format,
   grouped by commit type, scoped sections, links to compare URLs).
5. Creates the git tag (`v4.15.0`) and an empty release commit
   (`bump: version 4.14.0 → 4.15.0`).

Run `cz bump --changelog --check-consistency --yes` in CI on `main` and
the entire release is one job step — no hand-edited `CHANGELOG.md`
merge conflicts, no forgotten `__version__` constant, no "what was the
last version we shipped" archaeology in Slack.

## When to use

- **Multi-language monorepos that want one consistent commit + changelog
  + version-bump story.** `version_files` accepts arbitrary file:marker
  pairs, so the same `cz bump` updates a Python `pyproject.toml`, a Node
  `package.json`, and a Rust `Cargo.toml` in one transaction.
- **Teams adopting [`semantic-release`](https://semantic-release.gitbook.io/)
  without locking into Node.** commitizen is the Python-native answer to
  the same problem and reads exactly the same Conventional Commits
  vocabulary.
- **Projects already on [`pre-commit`](../pre-commit/) / [`lefthook`](../lefthook/).**
  Wire `cz check` as a `commit-msg` hook and a CI step in two lines
  total.
- **Anywhere an LLM-CLI workflow generates lots of commits.** The bot
  writes a Conventional Commit, `cz check` is the gate that catches
  malformed messages before they pollute the changelog.

## When NOT to use

- **Solo personal projects with no changelog and no semver discipline.**
  The investment doesn't pay back at one developer; `git log` is the
  changelog.
- **You want a *Node* tool.** Use the original
  `commitizen/cz-cli` + `cz-conventional-changelog` — same vocabulary,
  npm-shaped install, integrates more cleanly with `npm version` and
  `release-please`.
- **Your team rejects Conventional Commits as a format.** commitizen
  technically supports custom rule packs (`cz customize`) but the
  ecosystem around it (changelog grouping, semver inference) assumes
  the conventional shape. Fighting it is more work than adopting a
  different tool.
- **You need a release tool that *also* publishes** (uploads to PyPI /
  npm / crates.io, drafts GitHub Releases, runs CI gates). commitizen
  bumps + tags; pair with `goreleaser` / `hatch publish` / `npm publish`
  / [`cargo-release`](../cargo-release/) for the publish step.

## AI-native angle

- **The rule pack is the schema an LLM commit-author writes against.**
  Point a code-writing agent at the `cz_conventional_commits` spec (or a
  custom `cz customize` schema with your scopes enumerated) and the
  agent's commit messages become parsable, lint-passing input to
  `cz bump` — the agent's PRs are now release-ready, not message-edited
  before merge.
- **`cz bump --dry-run` is a release preview.** Wire it into a PR
  comment in CI and reviewers see "this PR causes a `minor` bump:
  4.14.0 → 4.15.0" before merge — the changelog is reviewable, not
  generated post-hoc.
- **Pairs with [`git-cliff`](../git-cliff/) for the changelog half.**
  git-cliff is the dedicated changelog generator; commitizen owns the
  *commit lint + version bump* half. Use both in shops where the
  changelog template is non-negotiable and `cz bump --changelog`'s
  default Keep-a-Changelog template doesn't fit.

## Alternatives in this catalog

- [`git-cliff`](../git-cliff/) — Rust changelog generator from
  Conventional Commits with a templated output. Orthogonal: git-cliff
  is *just* the changelog renderer; commitizen is commit lint + version
  bump + changelog. Use git-cliff when you want full control of the
  output template; use commitizen when you want the whole release driver
  in one command.
- [`cocogitto`](../cocogitto/) — Rust-implemented Conventional Commits
  toolkit (`cog commit` / `cog check` / `cog bump`) with the same
  three-job shape as commitizen but a single static binary. Pick
  cocogitto in a no-Python repo; pick commitizen in a Python project
  where `pyproject.toml` is the version source of truth and `pipx`
  is already on the box.
- [`convco`](../convco/) — focused Rust CLI for Conventional Commit
  validation + version-from-commits computation, no interactive
  composer, no changelog rewrite. Pick when you want the smallest
  binary that answers "what should the next version be" and you wire
  the changelog yourself.
- [`pre-commit`](../pre-commit/) / [`lefthook`](../lefthook/) — hook
  orchestrators that drive `cz check` as a `commit-msg` hook locally
  before CI sees the message.
- [`cargo-release`](../cargo-release/) — the *publish* half for Rust
  crates: bump + tag + push + `cargo publish` in one command.
  Complementary in polyglot repos (`cz bump` for the version /
  changelog / Python publish, `cargo-release` for the crate publish).
- [`gptcommit`](../gptcommit/) / [`opencommit`](../opencommit/) /
  [`aicommits`](../aicommits/) — LLM commit-message authors. Pair with
  `cz check`: the LLM proposes the message, commitizen enforces it
  matches the schema.
