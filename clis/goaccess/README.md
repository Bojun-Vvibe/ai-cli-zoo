# goaccess

## What it does
A **real-time terminal + HTML web log analyzer** in C: tail a web
server access log (NCSA Combined, Apache / Nginx custom formats,
Amazon CloudFront, AWS S3, AWS ELB, Google Cloud Storage, Caddy
JSON, traefik, Squid, W3C, syslog) and watch parsed metrics update
live in an `ncurses` dashboard, or emit a self-contained HTML report
with sortable tables and WebSocket-driven live updating, or dump
JSON / CSV for downstream pipelines. The output panels cover unique
visitors, requested files, static assets, 404s, hosts (with optional
GeoIP via libmaxminddb), operating systems, browsers, referrers,
referring sites, search keywords, HTTP status codes, virtual hosts,
and remote-user — all computed in-process against a custom
in-memory hash store (default) or a persistent on-disk store
(`--persist` with the Tokyocabinet-replacement BTree backend) so
multi-GB log dives don't blow RAM. Designed to run one-shot on a
log file, continuously on a `tail -F` pipe, or as a long-lived
daemon serving a live HTML report behind a reverse proxy.

## Why it's interesting
Different shape from `awstats` / Webalizer (Perl batch jobs that
emit static HTML on a cron — no live view, no terminal UI), from a
full ELK / Loki stack (powerful but requires a separate ingest
pipeline + storage tier + dashboard service for what is sometimes a
30-second question), from `lnav` (interactive log *viewer* with
SQL queries — different shape: lnav helps you read individual
lines, goaccess aggregates millions of lines into traffic metrics),
from Cloudflare / Fastly hosted analytics (vendor-locked, sampled,
no on-prem option), and from custom `awk` one-liners (no GeoIP, no
HTML report, falls over on rotated logs). goaccess is the *one C
binary, ncurses + WebSocket-live-HTML, no daemon required* shape:
pick it specifically when you have access logs on disk and want a
weekend of debugging a traffic spike — or a permanent live
dashboard pinned on a TV in the NOC — without standing up a search
cluster. Do **not** pick it for application-level metrics (use
Prometheus + Grafana), for full-text log search across many services
(use Loki / OpenSearch / [`lnav`](../lnav/) for single-host
exploration), or when you need long-horizon retention with
queryable history (the persistent store is for incremental updates,
not analytical queries).

## Niche category
Web log analyzer — single C binary that turns access logs into a
live ncurses dashboard or self-updating HTML report without a
search-cluster ingest pipeline.

## Repo
https://github.com/allinurl/goaccess

## Version pinned
`v1.9.4` (latest tagged release as of 2026-05-01)

## License
- SPDX: `MIT`
- License file in upstream repo: `COPYING`

## Install
```sh
# Homebrew (macOS / Linux)
brew install goaccess

# Debian / Ubuntu (official APT repo for latest, distro repo for older)
apt-get install goaccess

# Alpine
apk add goaccess

# From source (any POSIX)
curl -L https://tar.goaccess.io/goaccess-1.9.4.tar.gz | tar xz \
  && cd goaccess-1.9.4 && ./configure --enable-utf8 --enable-geoip=mmdb \
  && make && sudo make install
```

## Usage examples
```sh
# Interactive ncurses dashboard against an Nginx combined-format log
goaccess /var/log/nginx/access.log --log-format=COMBINED

# One-shot: emit a self-contained HTML report
goaccess access.log -o report.html --log-format=COMBINED

# Live HTML report with WebSocket updates as new lines land
goaccess access.log -o /var/www/html/stats.html \
  --log-format=COMBINED --real-time-html --ws-url=wss://stats.example.com:7890

# Pipe a tailed log into goaccess (rotation-safe with `tail -F`)
tail -F /var/log/caddy/access.log | goaccess - \
  --log-format=CADDY -o caddy.html --real-time-html

# Persistent on-disk store, incremental updates across runs
goaccess access.log --persist --restore --db-path=/var/lib/goaccess

# JSON output for piping into another analytics tool
goaccess access.log --log-format=COMBINED -o json | jq '.general'
```

## Date added
2026-05-01
