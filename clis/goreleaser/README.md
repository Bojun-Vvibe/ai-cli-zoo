# goreleaser

A release automation tool for Go (and, increasingly, Rust, Zig, and
generic binary) projects. Builds cross-platform artifacts, packages
them, signs them, generates SBOMs, and publishes to GitHub / GitLab /
Gitea releases, Homebrew taps, Scoop buckets, container registries,
and more — all from a single declarative `.goreleaser.yaml`.

## Repo

- URL: https://github.com/goreleaser/goreleaser

## Version

- v2.6.1 (OSS edition, 2025 release line)

## License

- MIT
- License file path in upstream: `LICENSE.md`

## Install

```sh
go install github.com/goreleaser/goreleaser/v2@latest
# or: brew install goreleaser
```

## Use case category

Release engineering / build & publish automation. Owns the "tag
pushed → artifacts published" pipeline so individual repos don't
need to hand-roll matrix builds.

## Example usage

```sh
# Validate the config and run a full release locally without publishing
goreleaser release --snapshot --clean

# In CI, on a tag push, do a real release
goreleaser release --clean
```

Expected behavior: cross-compiles binaries for every configured
`goos`/`goarch` pair, produces archives (tar.gz / zip), generates
checksums and an SBOM if configured, optionally signs with `cosign`,
builds Docker / OCI images, and uploads everything plus a generated
changelog to the configured release host. Exits non-zero on any
build, signing, or upload failure.

## Strengths

- One config replaces a sprawling matrix of CI scripts: cross-compile
  targets, archives, checksums, changelog generation, container
  images, package manager taps, and signing all live in one file.
- Great local UX — `--snapshot --clean` reproduces the full pipeline
  without touching any registry, which makes it easy to debug release
  problems before pushing a tag.
- Strong ecosystem of integrations: Homebrew, Scoop, Winget, nFPM
  for `.deb`/`.rpm`/`.apk`, Docker manifests, cosign keyless signing,
  and SBOM generation via `syft`.

## Limitations

- Heavily Go-centric in idioms; non-Go projects can use it via the
  `builds` `prebuilt` mode but lose some of the auto-detection magic.
- Some advanced features (notably nightly auto-publishing flows and
  certain SaaS integrations) are gated behind the paid Pro edition.

## Comparison

Versus `syft` (already in this zoo): `syft` produces an SBOM for an
existing artifact, while `goreleaser` orchestrates the whole release
pipeline that produces the artifact in the first place — and can call
out to `syft` and `cosign` as steps within that pipeline.

## When to reach for it

- You're shipping a Go (or prebuilt-binary) project and have outgrown
  hand-written `gh release create` shell scripts.
- You want reproducible local rehearsals of a release before pushing
  a tag, so contributors can debug release pipeline failures
  without burning real version numbers.
- You need consistent multi-channel publishing (Homebrew + Docker +
  `.deb`/`.rpm` + Winget) from one declarative config, and you want
  signing and SBOM generation wired in by configuration rather than
  by ad-hoc CI steps.
