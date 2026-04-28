# doggo

- **Repo:** https://github.com/mr-karan/doggo
- **Version:** v1.1.5 (latest stable, 2026-04)
- **License:** GPL-3.0 ([LICENSE](https://github.com/mr-karan/doggo/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install doggo` · `go install github.com/mr-karan/doggo/cmd/doggo@latest` · binary releases on the GitHub release page

## What it does

`doggo` is a modern command-line DNS client meant to replace `dig` for the
common "resolve this name, give me a readable answer" loop. It speaks plain
UDP/TCP, DNS-over-TLS (DoT), DNS-over-HTTPS (DoH), DNS-over-QUIC (DoQ), and
DNSCrypt — picked automatically from the resolver URL scheme (`tls://1.1.1.1`,
`https://dns.google/dns-query`, `quic://dns.adguard.com`,
`sdns://...`). Output is colourised, column-aligned, and switchable between
`pretty` / `json` / `yaml` / `short` via `--format`, so the same invocation
shells into a script (`doggo --short A example.com | head -1`) or a JSON
pipeline (`doggo --json MX example.com | jq '.responses[].answers[].rdata'`)
without re-learning a different tool. Reverse lookups are `--reverse`,
multiple names + types in one invocation are positional, EDNS subnet probing
is `--client-subnet`, and queries can be batched in parallel against multiple
resolvers in one command for "do these resolvers agree on this record" checks.

## When to pick it / when not to

Pick `doggo` when the daily loop is "tell me what this name resolves to,
ideally over my preferred encrypted resolver, and let me grep / jq the
result". The DoH/DoT/DoQ-out-of-the-box story is the differentiator over
`dig` — checking a record against `https://dns.google/dns-query` is one flag,
not a separate `kdig` install or a `curl` to the JSON API. The JSON output
mode makes it the right primitive for shell scripts that need to parse DNS
answers without `dig +short` / `awk` parsing fragility.

Skip it for deep DNS protocol debugging (DNSSEC chain validation, raw EDNS
option inspection, AXFR zone transfers, sub-millisecond timing of every
packet) where `dig +trace +dnssec +stats` and `kdig +bufsize=1232 +nsid` still
expose more knobs; pick `dig` for those. Skip it inside containers / images
that already ship `dig` and you only need one A record (the install cost is
not worth it). The GPL-3.0 license is fine for end-user CLI use but can be a
non-starter for vendoring the source into a closed-source product — pull from
the binary release in those cases. Pairs with [`dog`](../dog/) (Rust
alternative, MIT, similar UX, no DoQ) where licensing or feature delta
matters, and orthogonal to [`xh`](../xh/) / [`httpie`](../httpie/) which test
the HTTP layer that DNS resolution feeds into.

## Example invocations

```bash
# Default: resolve A records via the system resolver
doggo example.com

# Specific record types, multiple names, one invocation
doggo MX TXT example.com cloudflare.com

# Force DNS-over-HTTPS against Cloudflare, JSON output for jq
doggo A example.com @https://cloudflare-dns.com/dns-query --json | jq '.responses[].answers'

# DNS-over-TLS against Quad9, short output for scripting
doggo --short A example.com @tls://9.9.9.9

# Reverse lookup
doggo --reverse 1.1.1.1

# Compare answers from three resolvers in parallel
doggo A example.com @1.1.1.1 @8.8.8.8 @9.9.9.9
```
