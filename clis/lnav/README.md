# lnav

> **A SQL-aware log file navigator** — a single C++ TUI binary
> that auto-detects log formats, merges multiple files into one
> timestamp-sorted view, and lets you query the contents with
> SQLite. Pinned to **v0.14.0**
> ([LICENSE](https://github.com/tstack/lnav/blob/master/LICENSE),
> BSD-2-Clause).

Source: <https://github.com/tstack/lnav>

## Category

Log viewer / TUI. Replaces the `tail -f | grep | less` pipeline
that everyone reinvents during incident response.

## What it does

Point `lnav` at one or more log files (or a directory, or stdin)
and it auto-detects formats (syslog, nginx access, JSON-lines,
Python logging, Java stack traces, dpkg, strace, MongoDB JSON,
etc.), parses every line into typed fields, and presents a
single timestamp-merged tail-following view. You can then run
`;SELECT log_level, COUNT(*) FROM access_log GROUP BY 1` against
the parsed data live, set `:filter-out` regexes, jump by log
level, and pipe filtered output back to disk.

## Why it matters

Every developer ends up writing the same brittle pipeline:
`zcat *.log.gz | grep ERROR | awk ...`. `lnav` is the one tool
that knows your log format already, sorts multi-file timelines
correctly, and exposes the result as a queryable SQL table —
turning ad-hoc grep archaeology into a real analysis surface.
v0.14 added "log-oriented debugging" (jump from a log line to
the source file/line that emitted it).

## Install

```bash
# Homebrew
brew install lnav

# Debian / Ubuntu
apt install lnav

# Fedora
dnf install lnav

# Static build (any Linux)
curl -L https://github.com/tstack/lnav/releases/download/v0.14.0/lnav-0.14.0-linux-musl-x86_64.zip -o lnav.zip
unzip lnav.zip && sudo install lnav-*/lnav /usr/local/bin/

# Verify
lnav -V    # lnav 0.14.0
```

## License

BSD-2-Clause — see
[LICENSE](https://github.com/tstack/lnav/blob/master/LICENSE).
Permissive; binary redistribution requires preserving the
copyright notice.

## One Concrete Example

```bash
# point it at a directory of mixed logs — it auto-detects formats
# and merges them on timestamp
lnav /var/log

# tail multiple JSON-lines apps as one timeline
lnav app1/*.jsonl app2/*.jsonl

# inside the TUI:
#   :filter-out  HEAD /healthz
#   :hide-lines-before 10 minutes ago
#   ;SELECT cs_method, COUNT(*) c FROM access_log GROUP BY 1 ORDER BY c DESC
#   :write-csv-to /tmp/methods.csv

# headless / scriptable: run a query and dump CSV, no TUI
lnav -n -c ';SELECT log_level, COUNT(*) FROM syslog_log GROUP BY 1' \
     -c ':write-csv-to /dev/stdout' \
     /var/log/syslog

# use as a structured `tail -f` for an nginx access log,
# only ERROR-or-worse and only paths matching /api/
lnav -c ':set-min-log-level error' \
     -c ':filter-in /api/' \
     /var/log/nginx/access.log
```

## Niche It Fills

`tail -f` shows you bytes; `grep` matches lines; `jq` parses one
file; `awk` aggregates if you write the parser. `lnav` is the
only widely-installed tool that understands *log formats* as a
first-class concept — so `log_level`, `c_ip`, `cs_user_agent`
are typed columns you can `GROUP BY` instead of regex captures
you have to re-derive every incident. The killer feature is
running SQL against a live-tailing merged view of every log on
the box.
