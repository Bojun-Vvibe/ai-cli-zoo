# tspin

> **A log-file highlighter that wraps `less` and colorizes
> dates, IPs, UUIDs, URLs, numbers, key-value pairs, and log
> levels on the fly** — a single Rust binary (`tspin`,
> previously `tailspin`) that takes a path or a stdin pipe,
> applies a built-in regex ruleset for the common log shapes
> (syslog timestamps, RFC3339, ISO-8601, Apache / nginx access
> lines, JSON log records, Java stack traces, kubectl output),
> and pages the result through `less` so `q` quits, `/`
> searches, `G` jumps to tail, and `F` follows `tail -f`-style.
> Pinned to **v6.0.0** ([LICENCE](https://github.com/bensadeh/tailspin/blob/main/LICENCE),
> MIT).

Source: <https://github.com/bensadeh/tailspin>

## TL;DR

Reading raw logs in a terminal is the long-pole of debugging:
`cat app.log | less` is monochrome, `tail -f app.log` is
monochrome and disappears when you scroll, and rolling your own
`grep --color` rules per project is a habit that does not
survive switching laptops. `tspin` is the small fix: one binary
that knows the dozen regexes any sysadmin or backend dev would
have written by hand (timestamps, IP addresses, log levels,
HTTP status codes, UUIDs, paths, numbers, quoted strings, JSON
keys), wraps them around `less` so all of `less`'s navigation
keys work unchanged, and ships with a follow mode (`tspin -f`)
that behaves like `tail -f` but stays colored and stays
scrollable. v6.0.0 (2026-04) finalized the rename from
`tailspin` to `tspin` (the binary is now `tspin`, with a
compatibility `tailspin` symlink) and ships with a refreshed
default theme plus per-keyword highlight overrides via
`~/.config/tailspin/config.toml`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install tailspin

# Cargo
cargo install tailspin --locked

# Arch Linux
pacman -S tailspin               # extra repo
# or AUR: yay -S tailspin-bin

# Nix
nix-env -iA nixpkgs.tailspin

# Pre-built release tarball
curl -LO https://github.com/bensadeh/tailspin/releases/download/6.0.0/tailspin-aarch64-apple-darwin.tar.gz
tar xf tailspin-aarch64-apple-darwin.tar.gz
sudo install tspin /usr/local/bin/

# verify
tspin --version    # tspin 6.0.0
```

No config required for the default ruleset; drop a
`~/.config/tailspin/config.toml` to add or override keyword
groups (each group is a list of regexes + an ANSI style).

## Use it for

```bash
# Color a static log file and page it through less
tspin /var/log/syslog
tspin app.log

# Follow a growing log (like tail -f, but colored)
tspin -f app.log
tspin --follow /var/log/nginx/access.log

# Pipe arbitrary stdout through it (kubectl, journalctl, docker)
kubectl logs -f my-pod | tspin
journalctl -u nginx -f | tspin
docker logs -f my-container 2>&1 | tspin

# Pass extra flags through to less (mnemonic: anything after --)
tspin app.log -- -S         # don't wrap long lines
tspin app.log -- +G         # jump to end on open

# Print to stdout instead of paging (good for piping further)
tspin --print app.log | head
cat app.log | tspin --print | grep ERROR

# Use a custom config (extra regex groups, theme tweaks)
tspin --config-path ./my-tailspin.toml app.log
```

The mental model is "`less`, but your regex book is already
loaded". Because the underlying pager is real `less`, every
muscle-memory shortcut transfers: `/pattern` searches, `n` /
`N` step matches, `g` / `G` jump to top / tail, `F` enters
follow mode mid-session, `q` quits.

## Why include it in a CLI catalog

1. **It is the small dependency-free upgrade to an everyday
   workflow.** Every shell user reads logs; almost no one
   writes a `LESSOPEN` filter to color them. `tspin` is a one-
   line install (`brew install tailspin`) that turns
   `tail -f` into something you can actually scan, with no
   per-project setup and no editor / IDE in the loop.
2. **It composes with anything that writes to stdout.**
   `kubectl logs | tspin`, `docker logs | tspin`,
   `journalctl | tspin`, `terraform plan 2>&1 | tspin`,
   `cargo build 2>&1 | tspin` — whenever a tool emits
   timestamps / IPs / numbers / paths, `tspin` colors them
   without that tool needing to know `tspin` exists. That
   makes it useful in the same shell pipeline shape as
   [`bat`](../bat/) (file viewer) and [`delta`](../delta/)
   (git pager) — drop-in upgrades to an existing reader.
3. **Built-in `less` integration eliminates a category of
   shell aliases.** Without `tspin`, the colored-tail recipe
   is `grcat | less -R` (with `grc` config files) or a hand-
   rolled `awk` script piped through `less`. `tspin` collapses
   that to one binary and inherits `less`'s pager behaviour
   for free, including `F` follow-mode resumption — which
   `tail -f | grep --color` famously cannot do.

For an LLM-CLI workflow, `tspin --print` is the "plain
ANSI-colored output" mode an agent can capture into a buffer
or render in a transcript without `less` getting in the way;
piping `kubectl logs --since=1h | tspin --print` into a
prompt context preserves the highlighting hints (the agent
sees the same structure the human would).

## Vs Already Cataloged

- **Vs [`bat`](../bat/):** orthogonal — `bat` is a `cat`
  replacement that syntax-highlights *source code* (knows
  language grammars via `syntect`); `tspin` highlights *log
  output* (knows generic patterns: timestamps, IPs, levels).
  Use `bat` to read a `.py` file, use `tspin` to read its
  runtime log. They share the "pipe through `less`" trick but
  apply it to opposite content classes.
- **Vs [`delta`](../delta/):** orthogonal — `delta` colors
  unified diffs (git output, `diff -u`); `tspin` colors free-
  form log lines. They both wrap `less`; they do not overlap.
- **Vs [`lnav`](../lnav/):** closest peer — `lnav` is the
  full log-analysis TUI (it parses logs into a SQL-queryable
  view, knows about 30 log formats, has a histogram pane and
  a SQL prompt). `tspin` is a sliver of that surface: just
  the highlighting-and-paging part. Pick `lnav` when you want
  to ask "show me errors in the last 5 minutes grouped by
  source"; pick `tspin` when you want a colored `tail -f` and
  nothing else. `tspin` starts in milliseconds; `lnav`
  spends a second or two parsing.
- **Vs [`grc`](https://github.com/garabik/grc) (not
  cataloged):** older Python-based generic colorizer
  (`grcat`) with a config-file ecosystem; `tspin` is the
  modern Rust single-binary version with a built-in default
  ruleset and `less` follow-mode integration baked in. New
  users should pick `tspin`.
- **Vs [`humanlog`](../humanlog/):** orthogonal — `humanlog`
  is specifically a *structured-log pretty-printer* (it
  reformats JSON log records into a human-readable line
  format, optionally re-emitting JSON). `tspin` does not
  reformat; it only colors. Pipe `humanlog` first if your
  source emits JSON, then pipe into `tspin` for the timestamp
  / IP / level highlighting on top.

## Caveats

- **Requires `less`.** `tspin` shells out to `less` for the
  pager UI; if `less` is not on `PATH` (rare on Linux / macOS,
  occasional on minimal containers) the default mode fails.
  Workaround: `tspin --print` skips the pager entirely.
- **Built-in regex ruleset is opinionated, not configurable
  per-format.** It detects "things that look like
  timestamps / IPs / UUIDs / numbers" generically; it does
  not know "this is an Apache access log, parse fields". For
  field-aware parsing pick `lnav`.
- **Follow mode buffers; very high-rate logs can lag.**
  `tspin -f` reads incrementally and applies the ruleset
  before handing to `less +F`; for a log doing >50 MB/s you
  will see the highlighting fall behind raw `tail -f` output.
- **Binary renamed in v6.** v6.0.0 renamed the binary from
  `tailspin` to `tspin`; a compatibility symlink ships in
  the release tarball but distro packages may take a release
  cycle to follow. Scripts hard-coded to `tailspin` will
  keep working on most installs but should migrate.
- **Theme overrides are TOML, not a CLI flag.** To change a
  highlight color you edit `~/.config/tailspin/config.toml`;
  there is no `--color level=red` shortcut. The default theme
  is reasonable; most users never touch this.
