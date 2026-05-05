# dnslookup

> Snapshot date: 2026-05. Upstream: <https://github.com/ameshkov/dnslookup>

**A `dig` for the modern encrypted-DNS world.** `dnslookup` is a small Go
CLI that issues a single DNS query against a server you specify — but
unlike `dig`/`drill`, it speaks **DoH (DNS-over-HTTPS), DoT (DNS-over-TLS),
DoQ (DNS-over-QUIC), DNSCrypt, and plain UDP/TCP** through one uniform
URL-style argument. Point it at `https://dns.quad9.net/dns-query`,
`tls://1.1.1.1`, `quic://dns.adguard-dns.com`, or
`sdns://...` (a DNSCrypt stamp), and you get the same wire-level
answer dump back. Useful for verifying which encrypted resolver your
network actually reaches, debugging ECH/ECS behaviour, or scripting a
"is this DoH endpoint healthy?" check.

## Repo + version + license

- Repo: <https://github.com/ameshkov/dnslookup>
- Latest release: **`v1.11.2`** (2025-12-18)
- HEAD on `master`: `6921476`
- License: **MIT** —
  <https://github.com/ameshkov/dnslookup/blob/master/LICENSE>
- License path in repo: `LICENSE` (SPDX: `MIT`)
- Default branch: `master`
- Language: Go (built on `miekg/dns` + AdGuard's `dnsproxy` libraries)

## Install

```bash
# Homebrew
brew install ameshkov/tap/dnslookup

# Go
go install github.com/ameshkov/dnslookup@latest

# Or grab a static binary from the GitHub Releases page
# (linux/macOS/windows, amd64/arm64)
```

## Hello-world usage

```bash
# Plain UDP query against a public resolver (defaults to A record)
dnslookup example.com 1.1.1.1

# DNS-over-HTTPS against Quad9
dnslookup example.com https://dns.quad9.net/dns-query

# DNS-over-TLS against Cloudflare
dnslookup example.com tls://1.1.1.1

# DNS-over-QUIC against AdGuard
dnslookup example.com quic://dns.adguard-dns.com

# DNSCrypt via a stamp (sdns://) URI — discoverable via dnscrypt.info
dnslookup example.com sdns://AgcAAAAAAAAABzEuMC4...

# Ask for a specific RR type
RRTYPE=AAAA dnslookup example.com https://dns.quad9.net/dns-query

# Verbose mode (full request/response, TLS handshake details)
VERBOSE=1 dnslookup example.com tls://1.1.1.1
```

`dnslookup` is configured almost entirely through environment variables
(`RRTYPE`, `RRCLASS`, `INSECURE_SKIP_VERIFY`, `VERBOSE`, `TIMEOUT`,
`HTTP3`, `EDNS_SUBNET`, `PAD`, ...) rather than flags, which makes it
trivial to drop into a `Makefile` or a one-liner without quoting hell.

## Niche

The "**single binary, every DNS transport**" slot. The classic toolkit
splits along transport lines:

- [`drill`](../drill/) and `dig` — battle-tested for plain UDP/TCP,
  but DoH/DoT/DoQ support is patchy or absent depending on the build.
- [`doggo`](../doggo/) — the closest peer; modern Go CLI with DoH/DoT/DoQ
  and pretty output, oriented at humans browsing zones.
- [`kdig`](https://www.knot-dns.cz/) (Knot) — has DoT/DoH but ships as
  part of a larger DNS server distribution.
- `curl https://.../dns-query --data ...` — works for DoH only, painful
  for everything else.

`dnslookup` carves out the "I just need to verify *this exact server on
this exact transport* answers correctly" use case, with first-class
support for the obscure-but-important options: HTTP/3, EDNS Client
Subnet (`EDNS_SUBNET`), padding (`PAD`), DNSSEC (`DNSSEC=1`), and
DNSCrypt stamps. There is no interactive UI; output is one query, one
response, machine-friendly enough to grep.

## Why it matters

- **Encrypted-DNS is now the default** on iOS, Android, Firefox, Chrome,
  and most ad-blockers — but most diagnostics tooling still assumes
  port 53. `dnslookup` lets you reproduce what those clients do, against
  the exact endpoint they use.
- **One URL = one transport.** The transport is encoded in the scheme
  (`tls://`, `https://`, `quic://`, `sdns://`, bare host = UDP). No
  `--mode doh --no-verify --http3 --port 853` flag soup.
- **Stamps as first-class.** DNSCrypt and ODoH stamps (`sdns://`) work
  out of the box — handy when validating relays/anonymized DNS setups
  that `dig` simply can't speak.
- **Single static binary, MIT, ~10 MB.** Drops cleanly into a container
  health-check, a CI smoke test for your DoH endpoint, or a Tailscale
  exit-node verification script. No runtime, no config file required.
- **Same author as the AdGuard `dnsproxy` library** — so the protocol
  implementations are the production-grade ones used by AdGuard's own
  DNS infrastructure, not a hand-rolled side project.
