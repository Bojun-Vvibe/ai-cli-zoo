# slsa-verifier

> **The reference verifier for SLSA (Supply-chain Levels for
> Software Artifacts) provenance attestations — given a
> downloaded binary plus its `*.intoto.jsonl` provenance
> file (or a sigstore bundle, or a GitHub attestation), it
> checks the signature against the Sigstore transparency log
> (Rekor), verifies that the build ran in a trusted reusable
> workflow on a pinned source repo at a pinned tag, and
> exits non-zero if anything in the chain is wrong — turning
> "did this artifact really come from `org/repo` at
> `v1.2.3`'s tagged release workflow?" into a single
> deterministic CLI call suitable for CI gates and admission
> controllers.** Pinned to **v2.7.1** (released 2026-06-27),
> [LICENSE](https://github.com/slsa-framework/slsa-verifier/blob/v2.7.1/LICENSE),
> Apache-2.0.

Source: <https://github.com/slsa-framework/slsa-verifier>

## TL;DR

SLSA defines provenance as a signed in-toto attestation that
says "this artifact (sha256:...) was produced by *this*
builder running *this* workflow at *this* commit of *this*
source repo". `slsa-verifier` is the official tool that
checks all four of those claims for you, against:

- **Sigstore Rekor** — the public transparency log; the
  signature must have a verifiable inclusion proof.
- **Fulcio** — the OIDC-based CA that issued the signing
  cert; subject must match the trusted reusable workflow
  identity (e.g. `slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@refs/tags/v2.0.0`).
- **Source repo + ref** — extracted from the attestation
  payload, must equal what you pass on the command line.
- **Builder version pin** — only allow-listed builder
  workflows count as SLSA L3.

The three subcommands cover the universe:

- `verify-artifact <binary> --provenance-path <bundle> --source-uri github.com/org/repo --source-tag v1.2.3`
- `verify-image <oci-ref>` — same idea for container
  images (provenance fetched from the OCI registry).
- `verify-npm-package <tarball> --attestations-path <file> --package-name foo --package-version 1.2.3`

`verify-github-attestation` (added in v2.7.1) extends this
to GitHub-native attestations produced by `actions/attest-build-provenance`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install slsa-verifier

# Go install (any platform with Go ≥ 1.23)
go install github.com/slsa-framework/slsa-verifier/v2/cli/slsa-verifier@v2.7.1

# Static binary from GitHub Releases (linux / macOS / windows, amd64 / arm64)
curl -fsSL -o /usr/local/bin/slsa-verifier \
  https://github.com/slsa-framework/slsa-verifier/releases/download/v2.7.1/slsa-verifier-linux-amd64
chmod +x /usr/local/bin/slsa-verifier

# Verify (the verifier ships its own provenance — bootstrap by hash)
sha256sum /usr/local/bin/slsa-verifier
slsa-verifier version
```

## One Concrete Example

```bash
# Scenario: you downloaded `cosign-linux-amd64` v2.4.1 from sigstore/cosign
# releases and want CI to refuse to use it unless its SLSA L3 provenance
# checks out against the pinned reusable builder.

REPO=sigstore/cosign
TAG=v2.4.1
BIN=cosign-linux-amd64
PROV=cosign-linux-amd64.intoto.jsonl

curl -fsSLO https://github.com/$REPO/releases/download/$TAG/$BIN
curl -fsSLO https://github.com/$REPO/releases/download/$TAG/$PROV

slsa-verifier verify-artifact "$BIN" \
  --provenance-path "$PROV" \
  --source-uri "github.com/$REPO" \
  --source-tag  "$TAG"
# PASSED: SLSA verification passed
# echo $? -> 0 ; failure -> non-zero, CI fails the job

# Same idea for a container image — provenance lives in the OCI registry
slsa-verifier verify-image \
  ghcr.io/example/app@sha256:deadbeef... \
  --source-uri github.com/example/app \
  --source-tag v0.5.0

# Use as a CI gate (GitHub Actions step)
- name: Verify upstream tool provenance
  run: |
    slsa-verifier verify-artifact ./tools/$BIN \
      --provenance-path ./tools/$PROV \
      --source-uri github.com/$REPO \
      --source-tag  $TAG
```

## License

[Apache-2.0](https://github.com/slsa-framework/slsa-verifier/blob/v2.7.1/LICENSE),
SPDX `Apache-2.0`.

## Niche / positioning

The **policy-enforcement endpoint** of the SLSA chain.
Pick `slsa-verifier` when you have a downstream pipeline
that consumes upstream binaries / images / npm packages
and you want a deterministic, scriptable gate that proves
provenance — not just "is the signature valid" (that's
[`cosign verify`](../cosign/) / [`cosign verify-blob`](../cosign/))
but "does the signed claim match the source repo and tag I
expect, and was it built by a builder workflow I trust at
SLSA L3". Pick [`cosign`](../cosign/) when you only need
keyless signature verification without the structured
SLSA payload checks. Pick [`syft`](../syft/) +
[`grype`](../grype/) for SBOM generation and CVE scanning
— orthogonal concerns (what's *in* the artifact vs how it
was *built*). Pick [`witness`](https://github.com/in-toto/witness)
when you want to *create* in-toto attestations during your
own build (slsa-verifier only verifies). Skip when your
threat model doesn't include build-pipeline tampering — for
internal-only artifacts behind an authenticated registry,
`cosign verify` against your own KMS key is usually enough.
