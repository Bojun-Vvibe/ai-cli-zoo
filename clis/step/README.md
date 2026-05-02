# step

> **Zero-trust swiss-army knife for X.509, JWT, JWK, OAuth, OIDC,
> and SSH certificates** — a single Go binary that turns the
> arcane ceremony of "I need a cert / token / key for this thing"
> into one-liners. Pinned to **v0.30.2** (released 2026-03-22,
> [LICENSE](https://github.com/smallstep/cli/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/smallstep/cli>

## TL;DR

`step` is what you reach for when you need to *do* PKI rather
than read about it. It generates keys (`step crypto keypair`),
issues and inspects X.509 certs (`step certificate create`,
`step certificate inspect`), mints and verifies JWTs / JWS / JWE
(`step crypto jwt sign|verify`), drives OAuth / OIDC flows from
the terminal (`step oauth`), wraps the SSH CA workflow
(`step ssh certificate`), and pairs natively with `step-ca` (the
companion online CA) so a laptop, a CI runner, or a Kubernetes
pod can request a short-lived cert in one command. The whole
surface is scriptable, JSON-friendly, and uses sane defaults
(P-256, EdDSA, SHA-256) so you can stop copying OpenSSL
incantations from 2014.

## Install

```bash
# Homebrew (macOS / Linux)
brew install step

# Linux package managers
# Debian / Ubuntu: download .deb from releases
# Fedora / RHEL:  download .rpm from releases
# Arch:           pacman -S step-cli
# Nix:            nix-env -iA nixpkgs.step-cli

# Windows
# scoop install step
# winget install Smallstep.step

# Static binary (any OS, no deps)
# https://github.com/smallstep/cli/releases

# verify
step version    # Smallstep CLI/0.30.2 …
```

## Examples

```bash
# generate a P-256 keypair and a self-signed leaf cert in one go
step certificate create "example.local" example.crt example.key \
  --profile self-signed --subtle --no-password --insecure

# inspect any cert (PEM, DER, or live TLS endpoint)
step certificate inspect example.crt --short
step certificate inspect https://example.com --short

# mint a short-lived JWT signed by a key on disk
step crypto jwt sign \
  --key signing.key --iss me --aud you --sub user-42 --exp $(($(date +%s)+300)) \
  --subtle

# verify a JWT against a JWK set fetched over HTTPS
step crypto jwt verify --jwks https://example.com/.well-known/jwks.json \
  --iss https://example.com --aud my-api < token.jwt

# bootstrap against a step-ca and request a 24h cert (no human in the loop)
step ca bootstrap --ca-url https://ca.example.com --fingerprint <fp>
step ca certificate web.example.com web.crt web.key --not-after 24h
```

## Use when

- You want PKI primitives (cert, JWT, JWK, OAuth) as *commands*
  rather than as a thousand-line OpenSSL recipe.
- You run (or want to run) a private CA for service-to-service
  mTLS — `step` is the client to `step-ca` and handles automatic
  renewal via `step ca renew --daemon`.
- You need to debug "why is this cert / token rejected?" on the
  fly: `step certificate inspect`, `step crypto jwt inspect`, and
  `step certificate verify` give you a structured answer in one
  command.
- You want SSH certificates instead of `authorized_keys` sprawl —
  `step ssh certificate` issues short-lived user / host certs
  signed by a CA you control.

Skip `step` for "I just need one self-signed cert for localhost
once" — `mkcert` is a single command for that. `step` earns its
keep when the same workflow has to be repeatable, automatable,
and auditable across a fleet.
