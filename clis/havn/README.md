# havn

> **A fast, configurable port scanner with reasonable
> defaults** — a single static Rust binary that sweeps
> the standard service ports (or any custom set) on
> one or many hosts in a few seconds, with clean
> coloured output suitable for both humans and
> shell pipelines. Pinned to **v0.3.7**
> ([LICENSE](https://github.com/mrjackwills/havn/blob/main/LICENSE),
> MIT).

Source: <https://github.com/mrjackwills/havn>

## TL;DR

`havn` is what you reach for when you need a
quick "what's listening on this box?" without
spinning up `nmap`. It's an async Rust scanner built
on `tokio` that defaults to a curated list of ~1000
common service ports (the same set most people care
about: 22, 53, 80, 443, the database ports, the
container ports, the Kubernetes ports, etc.) and
prints results as they come in, not in a final
batch. The default behaviour is intentionally
constrained — TCP connect scans only, sane
concurrency caps, IPv4 + IPv6 dual-stack, no raw
sockets, no root needed, no SYN tricks, no OS
fingerprinting — which keeps it boring and safe to
drop into a Dockerfile or a CI smoke-test step. When
you do want to widen the aperture, flags let you
specify port ranges (`-p 1-65535`, `-p 22,80,443`),
multiple targets (`havn 10.0.0.1 10.0.0.2
example.com`), per-port timeouts, concurrency,
output format (default colour, `--json` for piping),
and silent mode for scripts. Output for each open
port includes the port number, the IANA service
name guess, and the connect time, so you can spot
"open but slow" services at a glance.

## Install

```bash
# Homebrew (macOS / Linux)
brew install havn

# Cargo
cargo install havn

# Docker (statically linked image — ~3 MB)
docker run --rm --network host mrjackwills/havn:latest 192.168.1.1

# pre-built binary (Linux x86_64 / aarch64 / armv6, Windows x86_64)
curl -L https://github.com/mrjackwills/havn/releases/latest/download/havn_linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin

# verify
havn --version    # havn 0.3.7
```

## Basic usage

```bash
# default scan — top common ports on a single host
havn 192.168.1.1

# scan a hostname (resolves both A and AAAA)
havn example.com

# scan multiple hosts in one invocation
havn 10.0.0.1 10.0.0.2 router.lan

# all 65k TCP ports (slower; the default set is enough for most cases)
havn -p 1-65535 192.168.1.1

# explicit port list
havn -p 22,80,443,3306,5432,6379 db.internal

# tighter timeout (default 2000 ms) for fast LANs
havn -t 500 192.168.1.0/24-style-list

# concurrency cap
havn -c 256 example.com

# JSON output for scripts / dashboards
havn --format json example.com | jq '.results[] | select(.open)'

# silent mode for CI smoke tests — exit 0 if any expected ports open
havn -p 80,443 -q example.com && echo "web up"
```

Configuration is entirely flag-driven; no config
file. The `--help` page is the canonical reference
and stays under one screen.

## When to choose

- **You want a 30-second sanity check of a box you
  just provisioned** — `havn host.example` returns
  the open ports faster than typing the equivalent
  `nmap` invocation, and the default port set covers
  the things you actually care about (SSH, web,
  databases, container runtimes, common dev servers).
- **You're scripting a CI smoke test** —
  `havn -p 80,443 -q $TARGET` exits cleanly with
  `--format json` for structured downstream
  consumers, no `nmap` XML parsing, no root
  required, single static binary you can drop into
  any container.
- **You want IPv6 to "just work"** — `havn` resolves
  AAAA records and scans both stacks by default, no
  separate flag needed. Useful on dual-stacked cloud
  hosts where half your services bind v4 and half
  bind v6.
- **You're behind tight egress rules** — TCP connect
  scans look like ordinary client connections to
  IDS/IPS, not like raw-socket SYN scans, so `havn`
  rarely trips network alerts that `nmap -sS` does.

## When NOT to choose

- **You need full nmap functionality** — service
  fingerprinting, NSE scripts, OS detection, UDP
  scans, ARP discovery, SCTP, IPv6 routing tricks —
  use [`nmap`](https://nmap.org/) or
  [`rustscan`](https://github.com/RustScan/RustScan).
  `havn` deliberately leaves all of that out.
- **You need to scan a /16 or larger** —
  `havn`'s sweet spot is one to a few dozen hosts.
  For internet-scale scans use
  [`masscan`](https://github.com/robertdavidgraham/masscan)
  or [`zmap`](https://github.com/zmap/zmap).
- **You need UDP coverage** — `havn` is
  TCP-connect-only by design. UDP "open" detection
  requires payload knowledge per service and is
  outside this tool's scope.
- **You want a TUI / interactive view** — `havn`
  is one-shot and stream-prints. For an
  always-on TUI of network activity reach for
  [`bandwhich`](../bandwhich/),
  [`sniffnet`](../sniffnet/), or
  [`bottom`](../bottom/).

## Why it fits the zoo

The zoo's network-tool cluster
([`bandwhich`](../bandwhich/),
[`sniffnet`](../sniffnet/),
[`gping`](../gping/),
[`xh`](../xh/),
[`dog`](../dog/),
[`doggo`](../doggo/),
[`trippy`](../trippy/)) collects
"single-static-binary takes on a Unix network tool
with sane defaults and colour output." `havn`
fills the port-scan slot with the same recipe: one
download, no runtime deps, no root, no surprising
flags, output that's readable by both humans and
`jq`. It complements `nmap` — which the zoo
deliberately doesn't try to replace — by being the
tool you actually invoke ten times a day for a
quick check, leaving `nmap` for the once-a-month
deep scan.

## Upstream pointers

- Repo: <https://github.com/mrjackwills/havn>
- Release notes: <https://github.com/mrjackwills/havn/releases>
- License: [MIT](https://github.com/mrjackwills/havn/blob/main/LICENSE)
- Changelog: <https://github.com/mrjackwills/havn/blob/main/CHANGELOG.md>
- Author: [@mrjackwills](https://github.com/mrjackwills)
  (also [`oxker`](../oxker/), a Docker container TUI
  already in the zoo).
