# cargo-release

> **One `cargo release` command that takes a Rust crate from "I'm ready
> to ship" to "tagged, pushed, and on crates.io" without a checklist.**
> A `cargo` subcommand that bumps the version in `Cargo.toml` (semver-
> aware: `cargo release patch` / `minor` / `major` / explicit `0.6.1` /
> `--bump pre-release`), updates dependent workspace crates that
> reference it via `path = "..."` plus `version = "..."`, regenerates
> `Cargo.lock`, runs an opt-in pre-release hook (`cargo test`,
> `cargo fmt --check`, custom command), creates a release commit, tags
> it (`v0.6.1` by default), pushes commits + tag, and runs
> `cargo publish` against crates.io — atomically, in one invocation
> across every member of a workspace, with `--execute` flipping it from
> dry-run-by-default to actually-do-it. Pinned to **v1.1.2** (released
> 2025, MIT OR Apache-2.0,
> [LICENSE-MIT](https://github.com/crate-ci/cargo-release/blob/master/LICENSE-MIT)
> /
> [LICENSE-APACHE](https://github.com/crate-ci/cargo-release/blob/master/LICENSE-APACHE)).

Source: <https://github.com/crate-ci/cargo-release>

## Repo

- URL: <https://github.com/crate-ci/cargo-release>
- Owner: crate-ci (community-maintained Rust release-engineering org;
  also home of `committed`, `typos`, `cargo-edit` — the same crowd that
  owns the Rust CI hygiene niche)
- License files:
  [LICENSE-MIT](https://github.com/crate-ci/cargo-release/blob/master/LICENSE-MIT)
  ·
  [LICENSE-APACHE](https://github.com/crate-ci/cargo-release/blob/master/LICENSE-APACHE)

## Version

`v1.1.2` — verify with `cargo release --version`. The dual-licence
file pair is the standard Rust ecosystem shape; pick either licence at
the consumer's option. Stable 1.x line, semver-respecting; defaults
trend conservative (dry-run by default since 0.21, no auto-publish
without explicit consent).

## License

**MIT OR Apache-2.0 (dual)** — permissive. The same dual-licence pair
the Rust standard library uses, so vendoring or embedding into any
Rust project is licence-compatible by construction.

## Install

- `cargo install cargo-release --locked` (canonical)
- `cargo binstall cargo-release` (pre-built binary via
  [`cargo-binstall`](../cargo-binstall/) — skips the ~30 s compile)
- `brew install cargo-release`
- pre-built tarballs on the
  [Releases page](https://github.com/crate-ci/cargo-release/releases)
  for Linux x86_64 / aarch64, macOS x86_64 / arm64, Windows x86_64

## What it does

`cargo release [LEVEL] [--execute]` is the whole release pipeline as
one command. By default it is a **dry run** that prints every step it
*would* take — only `--execute` (or `-x`) actually performs them. The
steps it owns:

1. **Version bump.** `cargo release patch` rewrites `version = "0.6.0"`
   → `version = "0.6.1"` in `Cargo.toml` (and every workspace member if
   `--workspace` / a workspace-wide config). `minor`, `major`, and
   explicit versions (`cargo release 0.7.0-rc.1`) all work; pre-release
   identifiers (`alpha.1`, `beta.2`, `rc.1`) and metadata
   (`+sha.abc1234`) are first-class.
2. **Workspace dependency rewrite.** If crate A in the workspace
   depends on crate B via `B = { path = "../b", version = "0.5" }`,
   bumping B's version updates A's `version = ` constraint to match —
   the published-on-crates.io version (the `version = ` field) and
   the path-dev version stay coherent. This is the bug class
   `cargo publish` will reject if you do it by hand and forget one.
3. **Lockfile refresh.** `cargo update --workspace` after the bump so
   `Cargo.lock` reflects the new version.
4. **Pre-release hook.** Configurable via `[workspace.metadata.release]
   pre-release-hook = ["cargo", "test", "--all-features"]` — runs
   arbitrary commands and aborts the release if they fail. Standard
   uses: `cargo test`, `cargo fmt --check`, `cargo clippy
   -- -D warnings`, `typos .`, `cargo audit`.
5. **CHANGELOG replacement.** `pre-release-replacements` rewrites
   placeholder strings in `CHANGELOG.md` (`{{version}}`, `{{date}}`,
   `{{tag_name}}`) so the unreleased section becomes a dated release
   section atomically with the bump.
6. **Release commit.** `chore: Release crate-name version 0.6.1` (the
   commit-message template is configurable).
7. **Tag.** `v0.6.1` by default; the `tag-prefix` and `tag-name`
   templates handle multi-crate workspaces (`crate-name-v0.6.1`) so
   `git describe` stays unambiguous.
8. **Push.** `git push --follow-tags` to the configured remote.
9. **Publish.** `cargo publish` against crates.io with the right
   ordering when multiple workspace crates depend on each other (B
   publishes before A, with a configurable settle delay so the index
   is searchable before A's publish queries it).

Every step is opt-out via flags (`--no-publish`, `--no-push`,
`--no-tag`, `--no-confirm`) and the `[workspace.metadata.release]` /
`[package.metadata.release]` table is the persistent config so a
maintainer's `cargo release minor -x` is a one-line release for the
whole org.

## When to use

- **Any crate that publishes to crates.io.** The 4-step manual recipe
  (`cargo set-version`, `git tag`, `git push --follow-tags`, `cargo
  publish`) survives the first ten releases and breaks on the
  eleventh — `cargo-release` collapses it to one command and never
  forgets a step.
- **Workspaces with internal dependencies.** The `path = ... version =
  ...` rewrite is the killer feature; doing it by hand is the most
  common cause of "publish failed because crate B is not yet on the
  index" errors.
- **CI-driven releases.** `cargo release --execute --no-confirm` in a
  job triggered by a `release/*` branch or a `[release]` commit
  trailer makes the human action "merge the PR", not "run the
  command".
- **Pair with [`git-cliff`](../git-cliff/) for the changelog.**
  Configure `pre-release-hook = ["git-cliff", "--tag", "{{tag_name}}",
  "-o", "CHANGELOG.md"]` and the changelog regenerates from
  Conventional Commits at every release.

## When NOT to use

- **You're not publishing to crates.io.** A binary-only project with
  no library crate doesn't need the `cargo publish` step — `cargo
  release --no-publish` works, but `cargo-dist` /
  [`goreleaser`](../goreleaser/) are better-shaped for "build cross-
  platform binaries, sign them, attach to a GitHub Release".
- **Polyglot monorepo where Rust is one of N languages.** Use
  [`commitizen`](../commitizen/) / `release-please` /
  [`cocogitto`](../cocogitto/) at the top level and let `cargo-release
  --no-tag --no-push --no-publish` handle just the version bump
  inside the Rust crates as a sub-step.
- **You don't trust automated `cargo publish`.** A misconfigured
  `[package.metadata.release]` can publish a crate before its
  dependents are ready. Start with `--no-publish` for the first few
  releases, audit the dry-run output (no `--execute`), then enable
  publish.
- **You want a generated GitHub Release with binary attachments.**
  Use [`goreleaser`](../goreleaser/) (Go-shaped but works on Rust
  binaries) or `cargo-dist`. cargo-release is about the *crate*
  publish, not the *binary distribution*.

## AI-native angle

- **The release is one command an agent can call.** `cargo release
  patch --execute --no-confirm` is the entire ship-it action; an LLM-
  driven release bot doesn't need to understand semver, lockfile
  refresh, tag naming, or workspace ordering — it calls one verb and
  the tool owns the rest.
- **Dry-run is a structured release plan.** Without `--execute`, the
  output enumerates every file change, command, tag, and publish in
  order — perfect input for a "preview the release in the PR
  comment" agent step (paste the dry-run output into the PR body for
  reviewer sign-off).
- **Pairs with [`commitizen`](../commitizen/) in polyglot monorepos.**
  Commitizen owns the commit-lint + cross-language version bump +
  changelog generation; cargo-release owns the *Rust publish* half
  with the workspace dependency rewrite the language-agnostic tools
  cannot do correctly.
- **Pairs with [`bencher`](../bencher/) for performance-gated
  releases.** Configure a `pre-release-hook` that runs the benchmark
  suite and uploads results to bencher; the release aborts if a
  threshold regresses, so "did this release regress p99 latency" is a
  CI assertion not a post-mortem.

## Alternatives in this catalog

- [`commitizen`](../commitizen/) — Python-implemented commit lint +
  version bump + changelog generator that works across `pyproject.toml`
  / `package.json` / `Cargo.toml` via `version_files`. Pick commitizen
  in a polyglot repo where Rust is one of N languages; pick
  cargo-release inside the Rust crate where the publish step + workspace
  dependency rewrite are the value.
- [`cocogitto`](../cocogitto/) — Rust Conventional Commits toolkit
  (`cog commit` / `cog check` / `cog bump`). Orthogonal: cocogitto
  owns the commit-vocab + changelog + version-bump half, cargo-release
  owns the crate-publish + tag + push half. Use both: cocogitto
  decides the bump, cargo-release executes it.
- [`git-cliff`](../git-cliff/) — Rust changelog generator from
  Conventional Commits. The conventional partner for a `pre-release-
  hook` to regenerate `CHANGELOG.md` before the release commit.
- [`goreleaser`](../goreleaser/) — declarative release pipeline for
  Go (now also Rust via `gobuild` → `cargo build`) with the binary-
  distribution half (cross-compile, archive, sign, SBOM, GitHub
  Release attachments) cargo-release deliberately omits.
- [`convco`](../convco/) — focused Rust CLI for Conventional Commit
  validation + next-version computation. Compose with cargo-release
  by passing `convco version --bump` output as the explicit version
  argument: `cargo release "$(convco version --bump)" --execute`.
- [`cargo-binstall`](../cargo-binstall/) — install pre-built
  cargo-release binaries instead of compiling from source. The pair
  most maintainers use for the install side.
- [`cargo-nextest`](../cargo-nextest/) — drop-in `cargo test`
  replacement; wire as the `pre-release-hook` instead of `cargo test`
  for ~3× faster gating runs.
