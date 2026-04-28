# jc

> **CLI tool and Python library that converts the output of dozens
> of common command-line programs, file-types and string formats
> into structured JSON, YAML, or Dictionaries** — pin point a
> single tool to the end of any UNIX pipeline and stop hand-rolling
> `awk`/`grep`/`sed` scrapers. Pinned to **v1.25.6** (released
> 2025-10-13, [LICENSE.md](https://github.com/kellyjonbrazil/jc/blob/master/LICENSE.md), MIT).

Source: <https://github.com/kellyjonbrazil/jc>

## TL;DR

`jc` is the answer when you want to feed `ifconfig` / `ps` /
`netstat` / `df` / `lsblk` / `mount` / `dig` / `route` / `who` /
`uptime` / `kubectl get` / `traceroute` / `iostat` / a syslog
line / `/etc/hosts` into `jq` and stop parsing whitespace by
hand. It ships ~120+ parsers covering the long tail of Linux,
macOS, BSD, and Windows utilities plus structured file formats
(INI, XML, YAML, CSV, KV, ASN.1, X.509 PEM, RFC-3339 timestamps).
Two call shapes:

```bash
# magic syntax — `jc` runs the command for you
jc ps -ef
jc dig example.com

# pipe syntax — `jc` reads stdin and you pick the parser
ps -ef | jc --ps
dig example.com | jc --dig
```

Either way the output is JSON you can pipe into `jq`, `mlr`, `fx`,
`jless`, or any HTTP body. Useful in scripts, CI, observability
agents, and anywhere a structured log beats a text scrape.

## Install

```bash
# Homebrew (macOS / Linux)
brew install jc

# pip
pip install jc

# Linux package managers
# Debian / Ubuntu: apt install jc
# Fedora: dnf install jc
# Arch: pacman -S jc
# Nix: nix-env -iA nixpkgs.jc

# verify
jc -v    # jc version 1.25.6
```

## Examples

```bash
# turn `ps` into JSON for jq
ps -ef | jc --ps | jq '.[] | select(.cpu_percent > 5) | .command'

# parse dig output into a structured answer set
dig +short example.com A | jc --dig | jq '.[0].answer'

# parse /etc/hosts in one line
jc --hosts < /etc/hosts | jq '.[] | select(.hostname[] | contains("local"))'

# run a command and JSON-ify the result in one shot
jc --pretty df -h
```

## Use when

- A shell pipeline ends in "now grep column 4 and hope nobody
  added a column" — replace it with `jc | jq`.
- Building observability sidecars or CI assertions that need
  structured fields out of legacy CLIs without writing parsers.
- Feeding traditional UNIX tool output into structured-log sinks
  (Loki, ELK, Splunk HEC, Vector) without a custom regex stage.
- An LLM/agent needs to *reason* over `kubectl describe` / `ps` /
  `dig` output instead of pattern-matching on whitespace.

Skip `jc` if your tool already has a `--json` / `-o json` flag
(use that — it's authoritative). `jc` shines on the older UNIX
utilities that will never grow one.
