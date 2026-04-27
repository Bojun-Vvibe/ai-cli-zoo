# dog

> **A modern `dig(1)` replacement** — colourful, JSON-emitting,
> DoH/DoT-capable command-line DNS client written in Rust.
> Pinned to **v0.1.0** (2020-11-07 release; `master` is the
> active line, build from source for the most recent fixes),
> [LICENCE](https://github.com/ogham/dog/blob/master/LICENCE),
> EUPL-1.2.

Source: <https://github.com/ogham/dog>

## TL;DR

`dig` is the DNS swiss army knife — and shows its 1990 BIND
heritage in every line of output: fixed-column tabular text
designed for terminals that did not yet have colour, mandatory
verbose headers (`;; ANSWER SECTION:`), and a flag surface where
`+short`, `+trace`, `+nsid`, `+norecurse` are mutually surprising.
`dog` is the rewrite for the modern shell: defaults to a coloured
table that fits on one screen, supports DNS-over-HTTPS
(`--https`) and DNS-over-TLS (`--tls`) without external tools,
emits JSON (`--json`) for piping into `jq`, accepts the record
type as a positional argument (`dog example.com MX` instead of
`dig example.com MX`), and queries multiple record types in a
single invocation (`dog example.com A AAAA MX TXT`). For 95% of
"what does this hostname resolve to" use cases it replaces a
multi-line dig command with a one-word query.

## Install

```bash
# Homebrew (macOS / Linux)
brew install dog

# Cargo (recommended for the most recent fixes — last release is 2020,
# but master has accumulated patches)
cargo install --locked --git https://github.com/ogham/dog

# Linux package managers
# Arch:   pacman -S dog
# Nix:    nix-env -iA nixpkgs.dog

# Windows
# scoop install dog

# release binary (Linux / macOS / Windows)
curl -Lo dog.zip "https://github.com/ogham/dog/releases/download/v0.1.0/dog-v0.1.0-x86_64-apple-darwin.zip"
unzip dog.zip && sudo install bin/dog /usr/local/bin/

# verify
dog --version    # dog v0.1.0
```

The 2020 release tag is the last tagged release; `master` has
~3 years of bugfixes including TLS cert validation and `EDNS(0)`
parsing improvements. Prefer `cargo install --git` over the
release binary for production use.

## License

EUPL-1.2 (European Union Public Licence) — see
[LICENCE](https://github.com/ogham/dog/blob/master/LICENCE).
Copyleft, OSI-approved, compatible with GPLv2/v3 and AGPL.
Distributing modifications requires source disclosure under
EUPL or a compatible licence; using the binary unchanged in any
context is unrestricted. Note the British spelling `LICENCE`
(not `LICENSE`) in the repo.

## One Concrete Example

```bash
# 1. The basic case — A records, defaults to your system resolver
dog example.com
# A example.com.        14400s    93.184.216.34

# 2. Multiple record types in one invocation (dig requires multiple calls)
dog example.com A AAAA MX TXT NS

# 3. Query a specific resolver — Cloudflare via UDP
dog @1.1.1.1 example.com

# 4. DNS-over-HTTPS to Cloudflare (no curl, no jq, no manual base64url)
dog --https @https://cloudflare-dns.com/dns-query example.com

# 5. DNS-over-TLS to Quad9 on port 853
dog --tls @9.9.9.9 example.com

# 6. JSON output for piping into jq (dig's +json flag does not exist)
dog --json example.com A AAAA MX | jq '.responses[0].answers[].data'

# 7. Reverse DNS — auto-detects PTR from an IP positional
dog 1.1.1.1
# PTR 1.1.1.1.in-addr.arpa.   1738s    one.one.one.one.

# 8. Pin TCP transport (useful when UDP is blocked or response > 512 bytes)
dog --tcp example.com TXT

# 9. Query a non-recursive nameserver directly (debugging delegation)
dog @a.iana-servers.net. example.com NS
```

## Niche It Fills

**The "modern DNS client" slot.** Three competitors:
`dig` (BIND, the canonical reference; verbose tabular output,
no DoH/DoT without external proxies), `kdig` (Knot DNS's CLI,
full DoH/DoT support, similar verbose output), and
`dnslookup` (ameshkov, Go, DoH/DoT/DoQ, JSON, but Go binary).
`dog` wins on (a) coloured table output that scans at a glance,
(b) querying multiple record types in one go, (c) Rust binary
with no runtime, (d) JSON output that round-trips through `jq`.
`kdig` wins when you need DNSSEC chain validation; `dnslookup`
wins when you need DNS-over-QUIC. For day-to-day DNS poking
`dog` is the friendliest.

## Why use it

1. **Multi-type queries in one call.** `dig example.com A` then
   `dig example.com AAAA` then `dig example.com MX` becomes
   `dog example.com A AAAA MX` — a single network round trip per
   type, output grouped by type, exit code aggregated. Matches
   how you actually investigate DNS ("what does this domain
   look like end-to-end").
2. **DoH / DoT first-class.** No `cloudflared`, no `stubby`, no
   `dnscrypt-proxy` — `dog --https @https://1.1.1.1/dns-query
   example.com` works out of the box. Critical when debugging
   DNS on a network that rewrites port 53 (hotel WiFi, captive
   portals, opinionated ISPs).
3. **JSON output.** `dog --json | jq` replaces parsing dig's
   tabular output with `awk` / `sed`. Stable schema, suitable
   for cron-driven monitoring.

For an LLM-CLI workflow `dog --json hostname A AAAA MX` is the
clean way to give a model the DNS state of a domain — structured,
parseable, no shell-quoting hazards.

## Vs Already Cataloged

- **Vs `dig`:** dog is the lighter-weight ergonomic replacement
  for ad-hoc lookups; dig is still the right answer for DNSSEC
  trace chains (`+trace +sigchase`), TSIG-authenticated queries,
  and any scenario where you need bug-for-bug compatibility with
  BIND. Keep both — they cost nothing to coexist.
- **Vs `nslookup`:** `nslookup` is the legacy interactive client
  shipped on every OS; its output format is unstable across
  platforms and its DoH/DoT support is nil. `dog` is what
  `nslookup` would be if rewritten in 2020.
- **Vs [`xh`](../xh/) for testing DoH manually:** you *can*
  hand-craft a DoH query with `xh -b https://1.1.1.1/dns-query`
  using base64url-encoded DNS wire format, but that requires
  building the query packet yourself. `dog --https` constructs
  the wire format, sets the right `Accept: application/dns-message`
  header, and decodes the response — three operations you would
  otherwise script.

## Caveats

- **Last tagged release is 2020.** `master` has ongoing fixes
  but new releases are infrequent. For long-running production
  monitoring prefer `kdig` (actively released by the Knot team)
  or `dnslookup` (steady release cadence).
- **No DNSSEC validation.** `dog` displays `RRSIG` records when
  asked but does not perform chain-of-trust validation. Use
  `delv` (BIND) or `kdig +dnssec +sigchase` when validation is
  required.
- **No DNS-over-QUIC (DoQ).** Only UDP, TCP, DoH, and DoT are
  supported; for DoQ reach for `dnslookup` or `q` (natesales/q).
- **No `+trace` mode.** `dig +trace` walks the delegation chain
  from the roots; `dog` only queries the resolver you point it
  at. For delegation debugging, dig or `drill` remain the right
  answer.
- **EUPL-1.2 is copyleft.** Linking the source into proprietary
  software requires legal review; using the unmodified binary
  is unrestricted. Most users only run the binary so this never
  comes up.
