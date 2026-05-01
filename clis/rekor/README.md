# rekor

## What it does
The Sigstore transparency-log CLI. `rekor-cli` uploads, queries, and verifies cryptographic records (signatures, attestations, in-toto statements, SBOMs, SSH and PGP signatures, hashedrekord entries) against a tamper-evident Merkle-tree log run by the public Sigstore instance at `rekor.sigstore.dev` — or against a self-hosted Rekor server. Each entry receives a signed, append-only inclusion proof so a verifier can later prove "this signature existed at this point in time" without trusting the issuer or the registry it lives in.

## Why it's interesting
Closes the supply-chain loop opened by `cosign` and `gitsign`: the signing tools push to Rekor; downstream verifiers (`cosign verify --certificate-identity ...`, policy controllers, admission webhooks) check the log for inclusion before trusting an artifact. Without Rekor, a leaked Fulcio cert is forever; with it, every signature is timestamped and publicly auditable, so a revocation policy can say "reject anything signed after time T". Pairs with `cosign`, `gitsign`, `oras`, and `in-toto` as the auditability backbone of the whole stack.

## Niche category
Software supply chain — transparency log client

## Repo
https://github.com/sigstore/rekor

## Version pinned
`v1.5.1`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS
brew install rekor-cli

# Go install
go install github.com/sigstore/rekor/cmd/rekor-cli@latest

# Or pre-built release binaries from
# https://github.com/sigstore/rekor/releases
```

## Usage examples
```sh
# Search the public log for everything signed by an identity
rekor-cli search --email alice@example.com

# Fetch a specific entry by UUID and verify its inclusion proof
rekor-cli get --uuid <uuid> --format json
rekor-cli verify --uuid <uuid>

# Look up every log entry that signed a given artifact (by sha256)
rekor-cli search --sha sha256:<digest>
```
