# gitsign

- **Repository:** https://github.com/sigstore/gitsign
- **Latest version:** v0.14.0
- **License:** Apache-2.0 — verified at [`LICENSE`](https://github.com/sigstore/gitsign/blob/main/LICENSE) (raw: https://raw.githubusercontent.com/sigstore/gitsign/main/LICENSE)
- **Niche:** Keyless git commit / tag signing via Sigstore (OIDC → ephemeral cert → transparency log)

## What it does

`gitsign` is a drop-in replacement for `gpg` as git's signing
backend that issues a **short-lived X.509 certificate** bound to
your OIDC identity (Google, GitHub, GitLab, your own
issuer) from Sigstore's Fulcio CA, signs the commit with the
ephemeral key, records the signature in the Rekor transparency log,
and throws the private key away. There is no long-lived key on disk
to lose, rotate, or have stolen.

```
git config --global gpg.x509.program gitsign
git config --global gpg.format x509
git config --global commit.gpgsign true
git config --global tag.gpgsign true

git commit -m "signed via OIDC, no GPG key"
gitsign verify --certificate-identity=you@example.com --certificate-oidc-issuer=https://accounts.example.com HEAD
```

`gitsign verify` resolves the signature back to (a) the OIDC
identity that authorized it, (b) the issuer that vouched for that
identity at signing time, and (c) the Rekor entry that proves the
signature existed at a specific timestamp.

## Why interesting

GPG-signed commits in practice are a long, sad list of failure
modes: the signing key is on the laptop the contributor lost three
years ago; the key was generated on a default-key-size that is now
considered weak; the key was never published to a keyserver; the
keyserver was deprecated; the user marked everything signed but the
verifier silently accepts unsigned merges; the org has no idea
*which* contributors hold valid keys.

`gitsign` deletes the key-management problem. The commit signature
encodes *the OIDC identity* (`you@example.com` from
`accounts.example.com`) at the moment of signing, the cert is valid
for ~10 minutes, and the proof lives in a public append-only log.
Verification policy becomes "the commit must be signed by an
identity in `@orgname` from `https://accounts.orgname.com`" — a
one-line CI check — instead of "the commit must be signed by a key
on this list of fingerprints we maintain in a wiki page".

It also composes with the rest of the Sigstore stack: the same
identity that signed your commit can sign your container images
(`cosign`), your release artifacts (`cosign attest` /
[`oras`](../oras/) attached signatures), and your SBOMs. One
identity, one log, one verify command shape.

## Pairs well with

- [`cosign`](../cosign/) — same OIDC → Fulcio → Rekor flow for
  container images and arbitrary OCI artifacts; `gitsign` is "the
  same idea but for git commits".
- [`oras`](../oras/) — when you want the signed-commit identity to
  also sign the build artifact pushed from that commit.
- [`gh`](../gh/) — GitHub already supports verifying gitsign
  signatures; `gh` shows them as "Verified" on PRs without any
  custom config on the consumer side.

## When to skip

- Air-gapped environments with no reachable OIDC issuer — gitsign
  needs to talk to Fulcio (or your private Fulcio) to mint the
  cert; long-lived offline keys via GPG / SSH-format git signing
  remain the only option.
- You explicitly need the *signer* to be a hardware key bound to a
  named human across years — gitsign's value proposition is the
  *opposite* (ephemeral cert + identity-in-log), and the two
  models don't really compose.
