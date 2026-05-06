# q

> **A tiny, ergonomic command-line DNS client that speaks UDP,
> TCP, DoT, DoH, DoQ and ODoH** — argument order is free
> (`q openai.com @1.1.1.1 AAAA` is identical to
> `q AAAA @1.1.1.1 openai.com`), output is colorised by default,
> and every modern encrypted-DNS transport is a one-flag switch
> rather than a separate binary.
> Pinned to **v0.19.12** (released 2026-02-27,
> [`gh api repos/natesales/q/releases/latest`](https://github.com/natesales/q/releases/latest),
> [LICENSE](https://github.com/natesales/q/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/natesales/q>

## TL;DR

The classic DNS clients each pick one slice of the problem and
stop there: **`dig`** (BIND) is the reference but its argument
grammar is positional and unforgiving (`dig @8.8.8.8 example.com
AAAA +tcp` versus `dig example.com @8.8.8.8 AAAA +tcp` — only
one of those is what you meant), it speaks plain Do53 only, and
DoH / DoT / DoQ require a different tool. **`drill`** (ldns) is
faster but inherits the same single-transport limitation.
**`kdig`** (Knot) speaks DoT and DoH but ships as part of the
Knot DNS server package. **`doggo`** ([already in this
catalog](../doggo/)) is the closest sibling: same "modern
encrypted DNS in one Go binary" niche, JSON output, but a
different ergonomic and feature surface. `q` collapses the
problem into a *very small* binary (~6 MB) where the CLI grammar
is order-free, every transport is a flag (`-u` UDP, `-t` TCP,
`--tls` DoT, `--https` DoH, `--quic` DoQ, `--odoh` Oblivious
DoH), output is colourised line-oriented text by default and
JSON via `--format=json`, and `~/.config/q.yml` lets you alias
resolvers (`q @cloudflare AAAA example.com` once you've defined
`cloudflare: https://1.1.1.1/dns-query`). The argument parser
auto-detects which token is the name, which is the type, and
which is the server (the `@` prefix is a hint, not a
requirement), so muscle-memory from `dig` keeps working while
typos stop being silent failures.

## Install

```bash
# Homebrew (macOS / Linux)
brew install natesales/repo/q

# Go install (any platform with Go 1.21+)
go install github.com/natesales/q@latest

# Pre-built binary from a release
curl -L \
  https://github.com/natesales/q/releases/download/v0.19.12/q_Linux_x86_64.tar.gz \
  | tar xz && sudo mv q /usr/local/bin/

# verify
q --version
```

## Representative examples

```bash
# 1. Plain Do53 over UDP — order-free arguments
q openai.com @1.1.1.1 AAAA
q AAAA @1.1.1.1 openai.com    # identical result

# 2. DoH (DNS-over-HTTPS) — one flag, full URL
q example.com --https https://1.1.1.1/dns-query

# 3. DoT (DNS-over-TLS) against a port-853 resolver
q example.com --tls 1.1.1.1

# 4. DoQ (DNS-over-QUIC) — newer transport, same CLI
q example.com --quic dns.adguard.com

# 5. Oblivious DoH (target + relay)
q example.com --odoh \
  --odoh-proxy https://odoh1.surfdomeinen.nl \
  --odoh-target https://odoh.cloudflare-dns.com

# 6. JSON output for piping into jq / log indexers
q example.com @1.1.1.1 ANY --format=json | jq '.answers'

# 7. Reverse lookup (PTR) — auto-detects the IP shape
q 1.1.1.1

# 8. Aliased resolvers from ~/.config/q.yml
#   resolvers:
#     cf: https://1.1.1.1/dns-query
#     quad9: tls://9.9.9.9
q @cf example.com AAAA
q @quad9 example.com MX
```

## When to use vs. alternatives

- Pick **q** when the workflow is "diagnose DNS from a laptop /
  jump host across a *mix* of plain, DoT, DoH, DoQ and ODoH
  resolvers" and one binary with order-free args + colourised
  output beats juggling `dig`, `kdig`, and `curl`.
- Pick **`dig`** (BIND) when canonical, byte-for-byte
  reference-implementation output is the deliverable (RFC
  diagnosis, ticket reproduction against an authoritative
  server, scripts in the wild that already parse `dig +short`).
  `dig` is the reference; `q` is the daily driver.
- Pick [`doggo`](../doggo/) for the closest sibling — same
  niche, also Go, also encrypted-DNS-first. `doggo` leans into
  *table* output (column-aligned by default) and a different
  flag spelling; `q` leans into *order-free args* and a smaller
  binary. Try both and pick the one whose default output your
  eyes parse faster.
- Pick [`dog`](../dog/) for a Rust take on the same
  "colourful modern dig" idea with a friendlier UX but a
  narrower transport story (Do53 / DoT / DoH; no DoQ / ODoH).
- Pick [`dnslookup`](../dnslookup/) (AdGuard) when the workflow
  is specifically scripting against AdGuard / NextDNS / pi-hole
  upstreams — same encrypted transport coverage, a more
  scripting-shaped CLI, less colourful output.
- Pick [`nexttrace`](../nexttrace/) for the *path* question
  ("which hops does my packet take and where does the latency
  land") — orthogonal verb. Compose: `q` resolves, `nexttrace`
  traces.
- Caveats: GPL-3.0 (note for vendoring inside permissively
  licensed binaries — invoke as a subprocess, don't link), the
  config file format is YAML and the schema is pre-1.0 (pin
  the version in CI rather than chasing `@latest`), and ODoH
  needs a *separate* relay URL — the flag exists but the
  ecosystem of public relays is small, so plan for the relay
  list to be the limiting factor in production use.
