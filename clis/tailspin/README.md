# tailspin

- **Repo:** https://github.com/bensadeh/tailspin
- **Version:** 6.0.0 (latest stable, April 2026)
- **License:** MIT ([LICENCE](https://github.com/bensadeh/tailspin/blob/main/LICENCE))
- **Language:** Rust
- **Install:** `brew install tailspin` · `cargo install tailspin` · `pacman -S tailspin` · `nix-env -iA nixpkgs.tailspin` · static binaries on the GitHub release page · binary name is `tspin` (formerly `spin`)

## What it does

`tailspin` is a log-file pager that **highlights structure without
parsing structure**. You pipe (or `tspin <file>`) any text log —
nginx access log, journald output, k8s pod logs, application
logs — and it auto-detects and colourises dates, timestamps,
IPv4/IPv6, UUIDs, file paths, URLs, HTTP status codes, key=value
pairs, JSON-ish blobs, log levels (`INFO`/`WARN`/`ERROR`),
numbers, and quoted strings. It uses `less` under the hood so you
get all of the familiar pager bindings (`/`, `?`, `n`, `N`, `g`,
`G`, `F` for follow), search highlighting, and incremental scroll.
It works on a static file, on a tail-followed file (`-f`), and as a
filter on stdin, which means it slots into any existing pipeline
in front of `less`, `bat`, or `jq` without configuration. Custom
highlight groups can be added via a TOML config when the built-in
heuristics aren't enough.

## When to pick it / when not to

Pick `tailspin` whenever you're reading more than a few hundred
lines of log output by eye and the log is plain text rather than
structured JSON. It's especially valuable for production
debugging — `kubectl logs -f pod | tspin`, `journalctl -u app -f
| tspin`, `tail -F /var/log/nginx/access.log | tspin` — because
the visual hierarchy makes it obvious at a glance where the
errors, the unusual status codes, and the slow-path latencies are.
It is also a good default `PAGER` substitute for log files because
it falls back gracefully on non-log content.

Skip it when your logs are **structured JSON** end-to-end — reach
for [`fx`](../fx/), [`jq`](../jq/), or [`jnv`](../jnv/) which
understand the schema and can filter / project. Skip it for
**multi-file aggregated tailing** — [`lnav`](../lnav/) is the right
tool, with cross-file timeline merging, SQL queries over log
contents, and format auto-detection. Skip it inside CI logs that
are already ANSI-coloured by the producer (you'll get double
colouring).

## Why it matters in an AI-native workflow

When an LLM coding agent reads a long log to diagnose a failure,
it spends most of its tokens identifying *where* the interesting
events are: which lines are errors, which lines are the
just-before-the-crash context, which timestamps cluster around the
incident. `tailspin` does that segmentation visually for the human
reviewer, and as a bonus its colour boundaries align with the
fields a regex-using agent would extract anyway, so a human and an
agent looking at the same buffer see the same structure. Because
it's a transparent stdin filter, it never alters the bytes the
agent captures with `tee` or `script`, so the highlighted view
and the analysed view stay in sync.

## Example invocations

```bash
# Page a static log file with auto-highlighting
tspin /var/log/syslog

# Follow a file like `tail -f` but with colour
tspin -f /var/log/nginx/access.log

# Filter on stdin from any producer
journalctl -u myapp -f | tspin
kubectl logs -f deploy/api -n prod | tspin
docker logs -f my-container 2>&1 | tspin

# Print to stdout instead of paging (good for piping into less / grep)
tspin --print /var/log/syslog | grep ERROR

# Disable the built-in pager and act as a pure stdin colouriser
cat app.log | tspin -p

# Use a custom config (override / add highlight groups)
tspin -c ~/.config/tailspin/highlights.toml app.log
```
