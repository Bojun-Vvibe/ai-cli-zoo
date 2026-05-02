# redli

- **Repo:** https://github.com/IBM-Cloud/redli
- **Version:** v0.17.0
- **License:** Apache-2.0 — [LICENSE.txt](https://github.com/IBM-Cloud/redli/blob/master/LICENSE.txt)
- **Category:** Redis client / TLS-aware interactive REPL

## What it is

`redli` is an alternative `redis-cli` written in Go, originally built
by the IBM Cloud Databases team to give managed-Redis users a CLI
that handles TLS, hosted-Redis URIs, and certificate bundles cleanly
out of the box. It implements the Redis RESP protocol, supports the
same interactive REPL feel as `redis-cli`, and adds quality-of-life
flags around connection strings and certificate handling.

## Why it's interesting

- **Native `rediss://` URI support.** A single
  `redli rediss://user:pass@host:6380/0` connects to a TLS-protected
  managed Redis (IBM Cloud, Aiven, Upstash, Redis Cloud, etc.) —
  no juggling `--tls --cacert --user --pass --port -h` flags the way
  upstream `redis-cli` historically required.
- **`--certfileb64` for CI / Kubernetes.** Cert chains can be passed
  inline as base64, which fits naturally into Kubernetes Secrets and
  CI variables; no temp file required.
- **Single static Go binary.** No OpenSSL link-time gymnastics — a
  ~10 MB binary drops onto Alpine, distroless, scratch, or a CI
  runner without dependencies, which makes it pleasant for
  smoke-test / health-check jobs against managed Redis.
- **Interactive REPL with history & completion.** Behaves like
  `redis-cli` for ad-hoc work (`KEYS`, `INFO`, `MONITOR`, `CLUSTER
  NODES`), but with sane defaults for hosted endpoints.
- **Useful defensive default.** Refuses dangerous commands like
  `FLUSHALL` against a TLS-protected managed endpoint unless
  explicitly confirmed — small but meaningful guardrail when
  ops-on-call shells in tired at 2 AM.

## Install

```bash
# macOS (Homebrew tap)
brew install IBM-Cloud/tap/redli

# Linux (binary release)
curl -sSL "https://github.com/IBM-Cloud/redli/releases/download/v0.17.0/redli_0.17.0_linux_amd64.tar.gz" \
  | tar -xz -C /tmp && sudo install /tmp/redli /usr/local/bin/redli

# From source
go install github.com/IBM-Cloud/redli@v0.17.0

# Verify
redli --version
```

## Quick start

```bash
# Connect to a managed Redis over TLS
redli -u rediss://default:$REDIS_PASSWORD@my-host.example.com:32123/0

# One-shot command
redli -u "$REDIS_URL" PING
redli -u "$REDIS_URL" CLUSTER NODES
```
