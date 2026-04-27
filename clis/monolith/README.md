# monolith

> **Save a complete web page as a single, self-contained HTML
> file** — CSS, JS, images, fonts, and iframes all inlined as
> base64 data URIs, no external requests on reopen, no broken
> assets six months later. Pinned to **v2.10.1** (2025-03-30
> release),
> [LICENSE](https://github.com/Y2Z/monolith/blob/master/LICENSE),
> CC0-1.0 (public domain).

Source: <https://github.com/Y2Z/monolith>

## TL;DR

`wget --page-requisites` and `curl -O` save the HTML and the
top-level assets but leave you with a directory tree of relative
URLs that breaks the moment you move it, archive it as a single
file, or open it from a different working directory. Browser
"Save Page As… (Complete)" produces a `.html` plus a sibling
`_files/` directory with the same fragility. `monolith` does the
one job neither does properly: walks the DOM, fetches every
referenced resource (stylesheets, scripts, images, fonts,
favicons, iframes, audio, video posters, SVG `<use>` targets),
and inlines each one as a `data:` URI inside the HTML. Output is
**one** file you can email, drop in S3, archive in `git`, render
in any browser offline forever. Optional flags strip JS
(`--no-js`), strip CSS (`--no-css`), strip images (`--no-images`),
isolate iframes, set a custom user-agent, follow `<noscript>`
content, or feed the page through a headless Chromium first
(`--unwrap-noscript` plus an external Chromium for SPA pages
that require JS to render their initial DOM).

## Install

```bash
# Homebrew (macOS / Linux)
brew install monolith

# Cargo
cargo install --locked monolith

# Linux package managers
# Arch:   pacman -S monolith
# Nix:    nix-env -iA nixpkgs.monolith
# Debian/Ubuntu: apt install monolith    # 24.04+

# Windows
# scoop install monolith
# winget install Y2Z.Monolith

# Docker
docker run --rm Y2Z/monolith https://example.com > example.html

# release binary (any OS / arch)
curl -Lo monolith \
  "https://github.com/Y2Z/monolith/releases/download/v2.10.1/monolith-aarch64-apple-darwin"
chmod +x monolith && sudo mv monolith /usr/local/bin/

# verify
monolith --version    # monolith 2.10.1
```

## License

CC0-1.0 — see
[LICENSE](https://github.com/Y2Z/monolith/blob/master/LICENSE).
Public-domain dedication; no attribution required, no patent
restrictions, the most permissive license available. Safe for any
downstream use.

## One Concrete Example

```bash
# 1. Archive a page as a single file
monolith https://example.com -o example.html

# 2. Strip JavaScript before inlining (safer for archival; many sites
#    embed analytics scripts that 404 once the vendor goes away)
monolith --no-js https://news.ycombinator.com/item?id=12345 -o hn-12345.html

# 3. Strip everything except the readable content (text + images)
monolith --no-js --no-css --no-fonts https://en.wikipedia.org/wiki/Rust_(programming_language) -o rust.html

# 4. Pipe an existing HTML file through monolith (resolves relative URLs
#    against -b base URL and inlines what they point to)
curl -s https://example.com | monolith - -b https://example.com -o example.html

# 5. Set a custom user-agent (some sites block default reqwest UA)
monolith -u 'Mozilla/5.0 (X11; Linux x86_64) Gecko/20100101 Firefox/127.0' \
  https://example.com -o example.html

# 6. Isolate iframes (drop them entirely instead of inlining — useful when
#    iframes load megabyte-sized ad payloads you do not want in the archive)
monolith --isolate https://example.com -o clean.html

# 7. Quiet mode + timeout for batch archival
monolith --silent --timeout 30 https://example.com -o example.html

# 8. Combine with `single-file-cli` chromium pipeline for SPA pages
#    (monolith alone cannot execute JS; pre-render with chrome-headless
#    if the page renders its content client-side)
google-chrome --headless --dump-dom https://spa.example.com \
  | monolith - -b https://spa.example.com -o spa.html
```

## Niche It Fills

**The "WARC for one page, in one file, no JVM, no
ArchiveTeam" slot.** Full-fidelity web archival has two
incumbent shapes: WARC files (`wget --warc-file`,
`browsertrix-crawler`, ArchiveTeam tooling) which are correct
but require a WARC reader (`pywb`, `replayweb.page`) to view; and
browser "Save Page As (Complete)" which is fragile and
multi-file. `monolith` occupies the third position: a single
self-contained `.html` you can `open` in any browser, attach to
an issue, drop in a bucket, or check into git. Not a substitute
for proper WARC archival of large crawls (no metadata, no
revisit records, no manifest), but the right tool for "save this
one article forever" or "snapshot this docs page before the
vendor rewrites it".

## Why use it

1. **One file, forever.** No `_files/` directory, no relative
   URLs, no asset rot. The output renders identically in 2026,
   2030, and 2040 from any browser, including from inside a
   `.zip` you never extracted.
2. **CC0 licence on the binary and the source.** No attribution
   requirement, no patent clauses, no copyleft surface. Embed
   the binary in commercial archival pipelines without legal
   review. (Few archival tools ship under CC0 — most are
   GPL/AGPL.)
3. **Composable with the rest of the rust-CLI ecosystem.** Reads
   stdin, writes stdout, exit codes are honest, no daemon. Pipe
   from `curl`, pipe to `bat` for inspection, drive from
   `xargs`, schedule from `cron` / `launchd` / `systemd`.

For an LLM-CLI workflow `monolith --no-js URL -o snapshot.html`
is how you give a model a *stable* version of a docs page or
RFC: the model can read it now, the test suite can read it next
year, and neither cares whether the upstream still exists.

## Vs Already Cataloged

- **Vs [`crawl4ai`](../crawl4ai/) /
  [`firecrawl`](../firecrawl/) / [`trafilatura`](../trafilatura/):**
  different goal. Those tools *extract* content (markdown, JSON,
  text) from web pages for LLM ingestion — they discard layout,
  CSS, fonts, and most images by design. `monolith` *preserves*
  the page exactly as rendered, for archival or visual reference.
  Pipe `monolith` output into `trafilatura --html` if you want
  both an archive and a clean text extract from the same fetch.
- **Vs `wget --page-requisites --convert-links`:** `wget`
  produces a directory tree with rewritten relative URLs;
  monolith produces one self-contained file. `wget` handles
  recursive site mirroring (which monolith does not); monolith
  handles single-page archival cleanly (which wget does not).
- **Vs browser "Save Page As (Complete)":** browser save
  produces `page.html` + `page_files/` and is fragile across
  moves. Browsers also do not let you script the save — monolith
  is a CLI you can put in `cron`.

## Caveats

- **No JavaScript execution.** `monolith` parses the static HTML
  it receives and inlines referenced assets; it does not run
  scripts. For SPA pages that render their initial DOM in JS
  (most modern web apps), pre-render with headless Chrome
  (`google-chrome --headless --dump-dom`) and pipe the output
  into `monolith -`.
- **Output files are large.** A typical news article is 50KB of
  HTML + 2-5MB of base64-encoded images and fonts. Add
  `--no-fonts --no-images` for text-only archives, or compress
  the output with `gzip` / `brotli` before storage.
- **No revisit / dedup like WARC.** Each archive is independent;
  if you snapshot the same site 100 times you get 100 copies of
  the favicon. For high-volume archival use `browsertrix-crawler`
  or `wget --warc-file` and serve via `pywb`.
- **`data:` URIs trigger CSP warnings.** Pages saved with
  `monolith` cannot be hosted under a strict Content-Security-
  Policy that disallows `data:` sources for scripts/styles.
  Open locally (`file://`) or relax CSP for the archive host.
- **Some sites block the default user-agent.** `reqwest`'s
  default UA is rejected by Cloudflare-fronted sites and a
  handful of CDNs; pass `-u "Mozilla/5.0 ..."` when you see
  403s.
