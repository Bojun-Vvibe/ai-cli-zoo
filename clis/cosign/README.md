# cosign

- **Repo**: https://github.com/sigstore/cosign
- **Version**: v3.0.6
- **License**: Apache-2.0 (`LICENSE`)

## What it is

A Sigstore tool for signing, verifying, and storing artifacts (container images, blobs, SBOMs, attestations) with support for keyless OIDC-based signing backed by the Fulcio CA and the Rekor transparency log.

## Why it's in the zoo

Container supply-chain security has converged on Sigstore, and cosign is the reference CLI. It removes the "where do I store the signing key" problem via short-lived certs tied to OIDC identity, and emits in-toto attestations that downstream admission controllers (Kyverno, policy-controller) can verify.

## Install

```sh
brew install cosign
```

## Quick example

```sh
cosign verify --certificate-identity-regexp '.*' --certificate-oidc-issuer-regexp '.*' ghcr.io/example/image:tag
```
