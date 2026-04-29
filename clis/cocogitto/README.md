# cocogitto

> **A Conventional-Commits-native git toolkit: enforces the spec at
> commit time via a `commit-msg` hook, walks history to compute the
> next semver bump from breaking-change footers, generates a
> changelog from a Tera template, and wraps `git tag` so a release is
> one `cog bump --auto` invocation that updates `CHANGELOG.md`,
> creates a signed tag, and runs configurable pre/post hooks.** Pinned
> to **7.0.0**, MIT
> ([LICENSE](https://github.com/cocogitto/cocogitto/blob/main/LICENSE)).

- **Repo:** https://github.com/cocogitto/cocogitto
- **Latest version:** 7.0.0
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `release-tooling` / `commit-discipline`
- **Language:** Rust
- **Install:** `brew install cocogitto` · `cargo install --locked
  cocogitto` · `pacman -S cocogitto` · prebuilt binaries on the
  GitHub release page · binary name is `cog`

## What it does

`cog` is the Rust answer to the patchwork of Node tools (`commitlint`,
`commitizen`, `standard-version`, `semantic-release`) that
collectively implement Conventional Commits + auto-versioning.
Everything runs from one binary and one `cog.toml`:

1. **`cog install-hook commit-msg`** drops a hook that rejects any
   commit whose subject does not match the Conventional Commits
   grammar (`<type>(<scope>)!: <description>`). Configurable types,
   scopes, and an optional `--no-verify`-resistant CI-side
   `cog check` that walks the commit range of a PR and fails on
   non-conforming commits.
2. **`cog commit feat auth "add OAuth flow"`** is the assisted-author
   path: type and scope as positional args, body and footers as
   prompts, the resulting commit is guaranteed to parse.
3. **`cog log`** filters `git log` by Conventional Commit type,
   scope, or breaking-change marker (`cog log --breaking-changes`),
   which is the single most useful release-prep query.
4. **`cog bump --auto`** is the release verb: scans commits since the
   last tag, computes the next semver (major if any `!:` or
   `BREAKING CHANGE:` footer, minor if any `feat`, patch otherwise),
   regenerates `CHANGELOG.md` from a Tera template, creates the
   annotated/signed tag, and runs configured `pre_bump_hooks` and
   `post_bump_hooks` (e.g. update `Cargo.toml` version, run tests,
   `cargo publish`, push to remote). Monorepo-aware via
   `[packages]` blocks that scope each package to a path and version
   it independently.
5. **`cog changelog`** standalone changelog generation in
   "remote-aware" mode (links commits and authors to GitHub /
   GitLab / Gitea / Bitbucket) for cases where you only want the
   text and not the tag.
6. **`cog verify`** validates a commit message string without
   touching git — the building block for editor integrations.

The GitHub Action `cocogitto/cocogitto-action` runs `cog check` on
PRs and `cog bump --auto` on merge to main, which is the canonical
"PR-gates-conventional-commits, merge-creates-release" loop.

## Why included

Conventional Commits is one of those conventions whose value is
entirely downstream: the commits themselves cost a few seconds each,
but the changelog, the semver bump, the release notes, the
"breaking change" search across history, and the LLM-friendly commit
log all depend on every commit conforming. The compliance must be
mechanical, not aspirational. `cog` provides exactly that: a
`commit-msg` hook locally, a `cog check` gate in CI, and a `cog bump`
verb that turns the resulting clean history into a release without
any further ceremony.

The catalog already ships [`git-cliff`](../git-cliff/) (Tera-templated
changelog generator), [`gptcommit`](../gptcommit/) and
[`opencommit`](../opencommit/) (LLM-assisted Conventional Commits at
write time). `cocogitto` is the *enforcement + release* piece of the
same workflow: the LLM-generated commits get validated by the hook,
the validated history gets rendered by either `cog changelog` or
`git-cliff` (`git-cliff` ships a Cocogitto preset for exactly this
reason), and `cog bump --auto` cuts the tag. Pick `cog` when you
want the full opinionated loop in one binary; pick `git-cliff` alone
when you only want the changelog and already have a release flow.

## Tradeoffs

- All-in-one means opinionated. The `cog.toml` schema is broad but
  the *workflow* is "Conventional Commits → semver → tag → push";
  if you need calendar versioning, train-based releases, or
  release-please-style PR-driven versioning, `cog` is not the fit.
- The `commit-msg` hook is bypassable with `git commit --no-verify`.
  The `cog check` CI gate is the actual enforcement; both are needed.
- Rust toolchain dependency for `cargo install`; the Homebrew bottle
  and prebuilt release binaries avoid this for end users but agents
  in fresh containers should prefer the prebuilt binary path.
- Monorepo support exists but is less mature than the single-package
  path; complex per-package release workflows may still want
  `release-please` or a custom script.
- MIT-only license (vs. the dual MIT/Apache common in the Rust
  ecosystem); a non-issue for most consumers.
