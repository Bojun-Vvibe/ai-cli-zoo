# git-changelog

> **git-changelog** — pawamoy/git-changelog, a Python CLI that
> renders a Markdown CHANGELOG straight from your git history by
> parsing Conventional Commits / Angular / Atom / Basic styles and
> grouping them under semver- or calver-bumped sections. Pinned to
> **2.9.4**, ISC — license file:
> [LICENSE](https://github.com/pawamoy/git-changelog/blob/main/LICENSE).

Source: <https://github.com/pawamoy/git-changelog>

## TL;DR

`git-changelog` reads `git log`, classifies each commit by its
prefix (`feat:`, `fix:`, `chore:` …) according to a chosen *style*,
detects which commits would trigger a major / minor / patch bump,
and writes a CHANGELOG.md grouped by version with sections for
Features / Bug Fixes / Code Refactoring / etc. The interesting
parts beyond "yet another conventional-commits parser" are:

- **In-place updates** — `git-changelog --in-place` only prepends
  the section for the *new* unreleased commits, so a long-lived
  CHANGELOG.md keeps its history without being rewritten on every
  run.
- **Bump detection + writing** — `--bump auto` figures out the
  next version from the commit set and inserts that header
  automatically; `--bump 2.0.0` lets you override; combined with
  `--release-notes` it can dump *only* the new section to stdout
  for a GitHub release body.
- **Jinja templates** — the rendered output is a Jinja template
  resolution, so a project can ship its own `keepachangelog.md.jinja`
  to match an existing CHANGELOG style instead of being forced into
  the default layout.
- **Provider links** — commit / PR / issue references are turned
  into Markdown links to GitHub, GitLab, Bitbucket, or a self-hosted
  Forgejo / Gitea, configured via `--provider`.

In CI it slots between `git push` and `gh release create`: parse the
commits since the last tag, render the section, hand the output to
`gh release create --notes-file -`.

## Install

```bash
# Pure-Python — install via pip / pipx / uv
# https://github.com/pawamoy/git-changelog/releases/tag/2.9.4

pipx install git-changelog==2.9.4
# or
uv tool install git-changelog==2.9.4
# or
pip install --user git-changelog==2.9.4
```

## Example commands

```bash
# Print the CHANGELOG to stdout (Conventional Commits style by default)
git-changelog

# Write/refresh CHANGELOG.md in place — only adds the new section
git-changelog --in-place --output CHANGELOG.md

# Auto-bump and stamp the next version header
git-changelog --in-place --bump auto

# Dump JUST the new release notes to stdout (for `gh release create`)
git-changelog --release-notes --bump auto > /tmp/notes.md
gh release create v$(git-changelog --bump auto --version) \
  --notes-file /tmp/notes.md

# Use a different commit style (Angular)
git-changelog --convention angular

# Render with a custom Jinja template
git-changelog --template path/to/keepachangelog.md.jinja
```

A minimal `pyproject.toml` config so `git-changelog` runs with no
flags:

```toml
[tool.git-changelog]
output = "CHANGELOG.md"
convention = "conventional"
template = "keepachangelog"
in-place = true
bump = "auto"
provider = "github"
```

## Niche it occupies

**Python-native CHANGELOG generator with in-place updates +
auto-bump + Jinja templating** — overlaps several existing entries,
each with a different bias:

- [`git-cliff`](../git-cliff/) — Rust-based, the most popular
  general-purpose CHANGELOG generator. `git-cliff` is faster,
  ships a static binary, and has a richer template DSL; pick
  `git-changelog` when you are already in a Python toolchain
  (pipx + pyproject.toml configuration in the same file as your
  build) and want one less binary to install in CI runners.
- [`cocogitto`](../cocogitto/) — Conventional Commits *enforcement*
  plus changelog + bump. Heavier opinion (also installs commit
  hooks, refuses non-conventional commits). `git-changelog` is
  read-only against history — it never refuses a commit, it just
  classifies what is there.
- [`convco`](../convco/) — minimal Conventional Commits validator
  + bump detector with no templating. Pick `convco` for a
  pre-commit hook; pick `git-changelog` for the actual rendered
  Markdown.
- [`commitizen`](../commitizen/) — interactive commit prompt +
  bump + changelog, again Python. `commitizen` is heavier and
  opinionated about *how* you commit; `git-changelog` only cares
  about the log it can read post-hoc, so it works even on a repo
  whose contributors do not all use a commit assistant.
- [`gitmoji-cli`](../gitmoji-cli/) — emoji-prefixed commit style.
  Complementary: `git-changelog --convention basic` will still
  group those commits sanely.
- [`lefthook`](../lefthook/) /
  [`pre-commit`](../pre-commit/) — hook runners that you would
  combine with one of the *enforcement* tools above; `git-changelog`
  itself does not need a hook.

Pairs cleanly with [`gh`](../gh/) for `gh release create
--notes-file`, with [`goreleaser`](../goreleaser/) (which can be
told to use an external CHANGELOG via `changelog.disable: true` and
shipping the `git-changelog` output as the release body), and with
[`hk`](../hk/) / [`mise`](../mise/) for pinning the exact CLI
version per repo.

Caveats: requires Python ≥3.9 in CI (vs `git-cliff`'s static
binary); the bump-detection rules are convention-driven — a repo
that mixes `feat:` and `feature:` arbitrarily will mis-bump until
the convention is normalised; the Jinja templating layer is power-
ful but means subtle template bugs can produce malformed Markdown
that only shows up at release time, so render to a temp file and
diff in CI before publishing.

## Citation

- Repo: <https://github.com/pawamoy/git-changelog>
- Latest release: **2.9.4**
- License: **ISC**
- License file: [LICENSE](https://github.com/pawamoy/git-changelog/blob/main/LICENSE)
