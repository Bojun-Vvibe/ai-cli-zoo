# convco

> **A Conventional Commits Swiss-army knife written in Rust —
> validates messages (`convco check`), interactively writes them
> (`convco commit`), computes the next SemVer bump from history
> (`convco version --bump`), and renders a CHANGELOG.md from the
> git log (`convco changelog`), all from one ~5 MB static
> binary.** Pinned to **v0.6.3**,
> [LICENSE](https://github.com/convco/convco/blob/main/LICENSE),
> MIT.

Source: <https://github.com/convco/convco>

## TL;DR

`convco` is the right tool when you have decided your repo will
follow [Conventional Commits](https://www.conventionalcommits.org/)
and you want one binary that handles the *entire* lifecycle:
write the commit, lint it on push, derive the next version from
the commit log, and emit a release-grade CHANGELOG. The
JavaScript reference stack (`commitlint`, `commitizen`,
`standard-version`, `conventional-changelog`) needs Node, four
`package.json` deps, and three config files; `convco` is a
single Rust binary and one `.versionrc` file. The headline
features — `convco version --bump` (parse history → SemVer next)
and `convco changelog` (git log → grouped Markdown) — are
deterministic, fast on big repos (10k+ commits in under a
second), and produce identical output across CI machines without
a Node lockfile to babysit.

## Install

```bash
# Homebrew (macOS / Linux)
brew install convco

# Cargo (any platform with a Rust toolchain)
cargo install convco --version 0.6.3

# Pre-built binary (macOS arm64 / x86_64 / Linux / Windows)
curl -L "https://github.com/convco/convco/releases/download/v0.6.3/convco-macos.zip" \
  -o convco.zip && unzip convco.zip && sudo mv convco /usr/local/bin/

# Docker
docker run --rm -v "$PWD:/repo" -w /repo convco/convco:0.6.3 version --bump

# Arch (AUR):  yay -S convco
# Nix:         nix-env -iA nixpkgs.convco

# verify
convco --version    # convco 0.6.3
```

## License

MIT — see
[LICENSE](https://github.com/convco/convco/blob/main/LICENSE).
Permissive, no copyleft, redistribute and embed in CI images
freely.

## One Concrete Example

```bash
# 1. interactive commit (replaces `git commit` for the day)
convco commit
# > pick type: feat / fix / docs / refactor / test / chore / ...
# > scope (optional): api
# > short description: add /v2/healthz endpoint
# > body / breaking change / closes …
# writes a fully-formed Conventional Commit and runs `git commit`

# 2. lint a single message in a pre-push or commit-msg hook
echo "feat(api): add healthz" | convco check --from-stdin
convco check                       # lint every commit on the current branch
convco check origin/main..HEAD     # lint only the PR's commits

# 3. compute the next SemVer from commit history
convco version                     # current: 1.4.0
convco version --bump              # 1.5.0   (one feat: in range)
convco version --bump --major      # 2.0.0   (forced)
NEXT=$(convco version --bump) && git tag -a "v$NEXT" -m "release v$NEXT"

# 4. generate / update CHANGELOG.md from git log
convco changelog > CHANGELOG.md
convco changelog --unreleased "$(convco version --bump)" > CHANGELOG.md

# 5. CI: fail the PR if any commit on the branch is non-conforming
convco check --from-rev origin/main || {
  echo "Non-conforming commit on branch — squash-merge or rewrite history."; exit 1;
}

# 6. project config — .versionrc to map types into changelog sections
cat > .versionrc <<'EOF'
{
  "host": "https://github.com",
  "owner": "acme",
  "repository": "widget",
  "types": [
    { "type": "feat",  "section": "Features" },
    { "type": "fix",   "section": "Bug Fixes" },
    { "type": "perf",  "section": "Performance" },
    { "type": "docs",  "hidden": true },
    { "type": "chore", "hidden": true }
  ]
}
EOF
```

## Niche It Fills

**One Rust binary that owns the full Conventional Commits
lifecycle (write → lint → version → changelog) without dragging
Node into your CI image.** The JS-native chain
(`commitizen` + `commitlint` + `husky` + `standard-version` +
`conventional-changelog`) is the de-facto standard but costs you
a `node_modules/` and four config files. `convco` collapses that
to one binary and one `.versionrc`. The space has competitors —
[`cocogitto`](../cocogitto/) (also Rust, also full lifecycle,
slightly more opinionated about workflow) and
[`git-cliff`](../git-cliff/) (Rust, *changelog only*, no commit
authoring or linting) — and `convco` is the right pick when you
want lifecycle parity with the JS stack but in one binary, with
a config format (`.versionrc`) that intentionally mirrors
`standard-version`'s so migration is mechanical.

## Why use it

Three things `convco` does that the JS reference stack does not:

1. **One binary, no Node.** `commitlint` + `commitizen` +
   `standard-version` is three npm packages, a `package.json`,
   a `husky` install step, and a Node version pin in CI.
   `convco` is `brew install convco` and you are done — your
   CI image stays lean and your pre-commit hook does not need
   a JS runtime to validate "did the dev type `feat:` correctly?"
2. **Deterministic CHANGELOG that survives history rewrites.**
   `convco changelog` parses the git log directly with no
   intermediate state file (no `.changelog-cache.json`), so a
   `git rebase -i` followed by re-running `convco changelog`
   produces a consistent result. `standard-version` writes
   commit hashes into the changelog and breaks on rewrite;
   `convco` lets you regenerate from scratch idempotently.
3. **`--from-stdin` and `--from-rev` make hook integration
   trivial.** A `commit-msg` hook is one line
   (`convco check --from-stdin < "$1"`); a pre-push lint of
   "every commit on this branch" is one line
   (`convco check origin/main..HEAD`). No config wrapper, no
   plugin loader, no glob expansion.

For an LLM-CLI workflow, `convco` is the right tool to gate an
agent's commits: pipe the proposed commit message through
`convco check --from-stdin`, reject and re-prompt on failure.
Pair with [`git-cliff`](../git-cliff/) if you only need the
changelog half and want richer templating; pair with
[`cocogitto`](../cocogitto/) if you also want enforced
branching/release ceremonies.

## Vs Already Cataloged

- **Vs [`cocogitto`](../cocogitto/):** the closest peer. Both
  are Rust, both cover write/lint/version/changelog. `cocogitto`
  is more opinionated — it ships a "monorepo packages" model,
  enforced GPG signing for tags, and a `cog` CLI that wants to
  own the release workflow. `convco` is lighter-touch:
  `.versionrc` mirrors `standard-version`, no enforced workflow.
  Pick `cocogitto` when you want the tool to drive the release
  process; pick `convco` when you want a calculator + linter
  you can drop into an existing process.
- **Vs [`git-cliff`](../git-cliff/):** orthogonal slicing.
  `git-cliff` is *only* the changelog generator (and the best
  one, with a Tera template engine for arbitrary output
  formats). `convco changelog` is fine for the default
  Conventional-Commits-grouped-by-type layout but cannot match
  `git-cliff`'s template flexibility. Use `convco` for
  lint+version+default-changelog; use `git-cliff` when the
  changelog format is the deliverable.
- **Vs [`gptcommit`](../gptcommit/) / [`opencommit`](../opencommit/)
  / [`aicommits`](../aicommits/):** orthogonal. Those are LLM
  *authors* of commit messages from a diff. `convco` is the
  *validator* and *consumer* of them. Stack them: LLM drafts,
  `convco check --from-stdin` validates, human approves.

## Caveats

- **Conventional Commits or nothing.** `convco` assumes the
  prefix grammar (`type(scope)?: subject\n\nbody\n\nfooter`).
  If your repo mixes conventions, `convco check` will be a
  noise machine. Either commit fully to the convention (and
  enforce in CI) or pick a different tool.
- **Pre-1.0 — small breaking changes still happen.** Output
  format of `convco changelog` has shifted across 0.x minor
  releases (footer rendering, breaking-change emoji); pin
  `convco` in CI and review the diff on upgrade.
- **No monorepo "package versioning" out of the box.**
  `convco version` reasons about *the repo*, not "this
  package versus that package within the repo." For
  per-package SemVer in a monorepo, reach for `cocogitto`'s
  packages mode or `release-please`.
- **No built-in commit signing / release publishing.** It
  computes the next version and writes the changelog; tagging,
  signing, and pushing the release are your shell loop's job.
  This is intentional (do one thing well) but means you write
  the release script yourself, e.g. `NEXT=$(convco version
  --bump) && git tag -s "v$NEXT" -m … && git push --tags`.
- **`.versionrc` not `convco.toml`.** The config file is
  JSON-shaped (mirrors `standard-version`), not TOML. Minor
  papercut for a Rust tool, but deliberate for migration
  compatibility.
