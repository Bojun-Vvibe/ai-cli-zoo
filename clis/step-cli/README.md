# step-cli

- **Repo:** https://github.com/smallstep/cli
- **Version:** v0.30.2
- **License:** [LICENSE](https://github.com/smallstep/cli/blob/master/LICENSE) (Apache-2.0)
- **Category:** PKI / X.509 + JWT + SSH crypto Swiss-army knife

## What it is

`step` is Smallstep's general-purpose crypto CLI — the user-facing companion
to `step-ca`. It packages X.509 certificate generation, CSR/cert inspection,
JWT/JWK/JWS/JWE signing and verification, OAuth/OIDC token retrieval, SSH CA
operations, and password/secret utilities into one Go binary with a uniform
`step <noun> <verb>` grammar. It is what you reach for instead of stitching
together `openssl x509`, `openssl req`, `jq`, `jose`, and a homemade ACME
client every time you need a cert or a signed JWT for a test.

## Install

```
brew install step                         # macOS / Linuxbrew
# or download the static binary from https://github.com/smallstep/cli/releases
step version
```

## Basic usage

```
step certificate create root.example "root.crt" "root.key" \
  --profile root-ca --no-password --insecure                       # offline root CA
step certificate create leaf.example "leaf.crt" "leaf.key" \
  --profile leaf --ca root.crt --ca-key root.key --no-password --insecure
step certificate inspect leaf.crt --short                          # human-readable
step certificate verify leaf.crt --roots root.crt                  # chain check
step certificate fingerprint root.crt                              # SHA-256 SPKI

step crypto jwk create pub.json key.json                           # generate a JWK
step crypto jwt sign --key key.json --iss me --aud you --sub demo --exp $(($(date +%s)+3600))
step crypto jwt verify --key pub.json --iss me --aud you

step ssh certificate alice@example.com alice-key                   # SSH user cert
step oauth --provider https://accounts.example.com --client-id ... # OIDC token
```

## When to use it

- You need to **mint, inspect, and verify X.509 certs** in scripts and CI
  without memorizing six different `openssl` subcommands.
- You want a **single CLI for JWT/JWS/JWE/JWK** that round-trips with the
  same key formats as the rest of your PKI.
- You operate an internal CA (`step-ca`) and want the matching client tool
  for ACME enrolment, SSH host/user certs, and short-lived workload identity.
- Skip it for **mass certificate issuance from public CAs** at the edge — a
  dedicated ACME client like `lego` is the better fit there; `step` shines
  when you control the trust roots.
