# lychee

> **A Rust async link checker that crawls Markdown / HTML / plain
> text / source files for URLs, validates them concurrently
> (HTTP / HTTPS / mailto / file / FTP), and emits human, JSON,
> Markdown, or SARIF reports — fast enough to run on every PR
> across docs sites, READMEs, and source trees.** Pinned to
> **v0.24.1** (homebrew),
> dual-licensed
> ([LICENSE-MIT](https://github.com/lycheeverse/lychee/blob/master/LICENSE-MIT)
> / [LICENSE-APACHE](https://github.com/lycheeverse/lychee/blob/master/LICENSE-APACHE)),
> MIT OR Apache-2.0.

Source: <https://github.com/lycheeverse/lychee>

## TL;DR

`lychee` is the link checker you reach for when `markdown-link-check`
is too slow and `htmltest` is too HTML-only. It walks any input
(globs, stdin, a remote URL, `**/*.md`, `**/*.html`, `**/*.rs`),
extracts every URL it finds (Markdown link syntax, HTML `<a href>`
/ `<img src>`, plain-text URLs in source comments, even
`Cargo.toml` / `package.json` repository fields), then validates
them in parallel with a configurable concurrency limit. The
checker handles HTTP redirects, retries with exponential backoff,
respects `robots.txt` (opt-in), supports HTTP basic auth and
custom headers, and can be configured to skip private / loopback
hosts so a CI run does not fail on `http://localhost`. Output
formats include human-readable summary, JSON for downstream
tooling, Markdown for posting to a PR comment, and SARIF for
GitHub Code Scanning. The `--cache` flag persists results in
`.lycheecache` so the next run only re-checks links that
changed status — repeat runs on a docs site are seconds, not
minutes. There is also an official
[lycheeverse/lychee-action](https://github.com/lycheeverse/lychee-action)
GitHub Action that wraps the binary.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lychee            # currently 0.24.1

# Cargo (any platform with a Rust toolchain)
cargo install lychee

# Pre-built binaries + Docker image
# https://github.com/lycheeverse/lychee/releases
# docker pull lycheeverse/lychee

# Verify
lychee --version               # lychee 0.24.1
```

## License

Dual MIT OR Apache-2.0 — see
[LICENSE-MIT](https://github.com/lycheeverse/lychee/blob/master/LICENSE-MIT)
and
[LICENSE-APACHE](https://github.com/lycheeverse/lychee/blob/master/LICENSE-APACHE).
Permissive: vendor, fork, ship inside a commercial product
without copyleft obligations; the Apache option adds an
explicit patent grant.

## One Concrete Example

```bash
# 1. Check every Markdown file in the repo, cache results, fail on
#    broken links — one-line CI gate.
lychee --cache --max-cache-age 1d './**/*.md'

# 2. Check a live docs site (recursive crawl, depth 2, skip
#    fragment-only links).
lychee --max-concurrency 16 --no-progress \
       --exclude-all-private \
       https://example.com

# 3. Emit Markdown summary, post as a PR comment in CI.
lychee --format markdown --output ./link-report.md \
       --no-progress \
       'docs/**/*.md' 'README.md' 'CHANGELOG.md'

# 4. SARIF output for GitHub Code Scanning.
lychee --format sarif --output lychee.sarif './**/*.md'

# 5. Per-project config (`lychee.toml`) — exclude flaky hosts,
#    set timeouts, set headers.
cat > lychee.toml <<'EOF'
max_concurrency = 16
timeout = 20
retry_wait_time = 2
max_retries = 3
accept = [200, 206, 429]            # tolerate Cloudflare 429
exclude = [
  '^https?://localhost',
  '^https?://127\.0\.0\.1',
  '^https?://example\.invalid',
]
exclude_path = ['./vendor', './node_modules']
cache = true
EOF
lychee .
```

## Niche It Fills

**The fast, format-agnostic, async link checker.** Same family
as `markdown-link-check` (Node), `htmltest` (Go, HTML-only),
`muffet` (Go, site crawler), `linkchecker` (Python) — `lychee`'s
specific corner is *one Rust binary, async-first concurrency,
ingests anything (Markdown / HTML / plain text / source), emits
JSON + Markdown + SARIF, with persistent cache for fast repeat
runs*. Pick when you need a single link checker that covers
both "lint the README before merge" and "crawl the published
docs site nightly," or when you want SARIF output that drops
straight into GitHub Code Scanning.

## Why use it

Three things `lychee` does that pay off immediately:

1. **Async + concurrency limit + cache.** A first run on a
   500-link docs tree completes in seconds; subsequent runs
   only re-check links whose `.lycheecache` entry expired or
   changed status. CI cost is near-zero on a clean repo.
2. **Ingest-anywhere.** Same binary checks `**/*.md`,
   `**/*.html`, raw source files (URLs in `//` comments
   count), `package.json`, a remote site by URL, or stdin.
   No "use tool A for docs and tool B for the published site."
3. **First-class machine output.** `--format json` for piping
   into agents and dashboards; `--format markdown` for posting
   to PR comments; `--format sarif` for GitHub Code Scanning.
   The schema is documented and stable enough to script
   against.

For an LLM-CLI workflow, `lychee --format json` is the right
input to feed an agent that auto-rewrites broken links —
each entry includes URL, status code, source file, and line
number, so the agent can do targeted edits instead of
re-grepping.

## Vs Already Cataloged

- **Vs [`vale`](../vale/):** `vale` is a prose linter
  (style, terminology, grammar); `lychee` validates URLs.
  Orthogonal — run both in CI on the docs tree.
- **Vs [`harper`](../harper/):** `harper` is a grammar
  checker for source code and Markdown — also orthogonal,
  also runs alongside `lychee`.
- **Vs [`vacuum`](../vacuum/):** `vacuum` lints OpenAPI specs;
  `lychee` checks links inside any text. Different domain.
- **Vs [`markdownlint-cli2`](../markdownlint-cli2/):**
  `markdownlint-cli2` enforces Markdown style (heading levels,
  list indentation, line length); it does *not* validate that
  links resolve. Pair with `lychee` for full Markdown CI:
  `markdownlint-cli2` for structure, `lychee` for liveness.

## Caveats

- **Async means rate-limit pressure.** Default concurrency
  (128) will trip rate limits on GitHub, Cloudflare, and many
  CDNs. Set `--max-concurrency 16` or lower for any host you
  hit repeatedly, and add `accept = [429]` if you tolerate
  rate-limit responses as "not broken."
- **`.lycheecache` is per-checkout.** CI runners that wipe the
  workspace between runs lose the cache; mount it as a CI
  cache (`actions/cache` keyed on the cache file's hash) or
  accept full re-runs.
- **Crawl mode is shallow.** `lychee` extracts links and
  validates them, but it is not a full site crawler with
  JavaScript execution — single-page apps that render links
  client-side will return mostly empty results. For SPA docs
  sites, render to static HTML first.
- **Exit code 0 on warnings.** By default lychee exits non-zero
  only on hard failures (broken / 4xx / 5xx). Use
  `--require-https` and similar flags to upgrade warnings to
  failures if you want a strict gate.
- **No JS execution.** `lychee` issues HTTP requests; it does
  not run a headless browser. Links served only after a
  client-side route mount are invisible to it.
