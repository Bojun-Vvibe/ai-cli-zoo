# trafilatura

> Snapshot date: 2026-04. Upstream: <https://github.com/adbar/trafilatura>

"**Web → clean text + metadata, in one Python call or one shell
command.**" `trafilatura` is a focused web-scraping / extraction
toolkit purpose-built for the LLM-data-prep step that everyone
re-invents badly with `curl | html2text` or BeautifulSoup soup.
Given a URL or raw HTML, it discriminates the *main content*
(article body) from boilerplate (nav, sidebars, footers, ads,
comment threads, cookie banners, share widgets) using a tuned
combination of trafilatura's own algorithm plus optional fallbacks
to `readability-lxml` and `justext`, then emits the result as
plain text, Markdown, JSON, CSV, XML, or TEI-XML — together with
extracted metadata (title, author, date, hostname, categories,
tags, language, fingerprint). It also ships a sitemap / feed
discovery layer and a focused crawler so you can go from "this
site" to "every article on this site, deduplicated, language-
filtered, in JSONL" without touching `scrapy`.

## 1. Install footprint

- `pip install trafilatura` (Python ≥ 3.9). Pulls a small,
  carefully chosen dep set: `lxml`, `lxml-html-clean`, `urllib3`,
  `certifi`, `htmldate`, `courlan`, `justext`, `charset-normalizer`.
  No browser, no headless Chromium — pure parser stack, fast and
  cheap to install on CI runners.
- Optional extras: `pip install trafilatura[all]` adds language
  detection (`py3langid` / `cld3`), faster JSON (`orjson`),
  Brotli + zstd HTTP decoding, and `pycurl` for parallel fetches.
- Single binary on PATH after install: `trafilatura` (also
  `trafilatura --help` → subcommand surface for crawl / sitemap /
  spider).
- Zero JS execution by default — if the page is a SPA that needs
  Playwright, reach for [`crawl4ai`](../crawl4ai/) instead.

## 2. Repo + version + license

- Repo: <https://github.com/adbar/trafilatura>
- Latest release: **v2.0.0** (2024-12-03)
- License: **Apache-2.0** —
  <https://github.com/adbar/trafilatura/blob/master/LICENSE>
- Default branch: `master`
- Language: Python (with a thin shell front-end)

## 3. Inputs / outputs supported

- **Inputs**: HTTP / HTTPS URL, local HTML file, raw HTML string
  on stdin, sitemap.xml, RSS / Atom feed, plain text URL list,
  arbitrary seed URL with `--crawl` for in-domain BFS.
- **Outputs**: `txt` (default), `markdown`, `json`, `csv`,
  `xml`, `xmltei` (TEI-XML for digital-humanities corpora), and
  `html` (cleaned-but-still-HTML). Each carries the same metadata
  block (title, author, date, hostname, sitename, categories,
  tags, language, license, ID, fingerprint).
- **Metadata extractors**: `htmldate` for publication / modified
  date with confidence scoring, JSON-LD / OpenGraph / microdata
  / meta-tag fallbacks for author + title.
- **Language filters**: `--target-language en` drops pages whose
  detected language doesn't match — critical when crawling
  multilingual sites for a single-language corpus.
- **Deduplication**: `--no-dedup` to disable; the default uses a
  rolling hash to skip near-duplicates within a session.

## 4. When to use it

- You want to **build an LLM training / RAG corpus from the open
  web** and you need clean Markdown without nav-bar pollution
  inflating the token bill 3–10×.
- You want a **fast, headless-browser-free** extractor that can
  process tens of thousands of pages on a small CI runner —
  trafilatura's pure-`lxml` path benchmarks substantially faster
  than Playwright-based extractors and cheaper than LLM-based
  cleaners like `firecrawl`.
- You need **publication metadata** (date, author, language) per
  document for downstream filtering / sorting / citation, not just
  body text.
- You want **shell-pipeable extraction**: `trafilatura -u
  https://example.com/post --output-format markdown > post.md` is
  a one-liner you can paste into a Makefile or `xargs` over a URL
  list.
- You need a **sitemap / feed crawler** to discover URLs first:
  `trafilatura --sitemap https://example.com/sitemap.xml --list`
  emits one URL per line; pipe to `xargs -P 8 trafilatura -u`.

## 5. When NOT to use it

- The page is a **JavaScript SPA** that renders content client-
  side — trafilatura sees the empty shell. Use
  [`crawl4ai`](../crawl4ai/) (Playwright-backed, MCP-server-able)
  or a headless-browser stack instead.
- You want **typed structured extraction** (pull `price`, `rating`,
  `review_count` into a JSON schema) — trafilatura emits *body
  text + standard metadata* only. Layer an LLM extractor on top,
  or use [`crawl4ai`](../crawl4ai/)'s `-s schema.json` mode, or
  reach for `scrapy` + `pydantic`.
- You need **PDF / DOCX / PPTX → text** — that's
  [`docling`](../docling/), [`marker`](../marker/), or
  `pypdf`. Trafilatura is HTML-only.
- You want a **chat-style "fetch this URL" tool inside an agent
  CLI** — wire [`crawl4ai`](../crawl4ai/) as an MCP server
  instead; trafilatura has no MCP surface.
- You want **stealth crawling** (rotating proxies, fingerprint
  randomisation, CAPTCHA solving) — pick `scrapy` + middleware
  or a hosted scraping API.

## 6. Closest alternatives

- [`crawl4ai`](../crawl4ai/) — Playwright-backed, handles SPAs,
  doubles as MCP server. Heavier (~300 MB browser binaries) but
  necessary when JS rendering matters.
- [`docling`](../docling/) / [`marker`](../marker/) — document-to-
  Markdown converters for PDF / DOCX / PPTX; pair with trafilatura
  for the "anything → Markdown" pipeline (HTML through trafilatura,
  binaries through docling / marker).
- `readability-lxml` / `justext` / `newspaper3k` — narrower main-
  content extractors; trafilatura wraps the first two as optional
  fallback extractors (`--fallback`) and benchmarks favourably
  against all three on the published evaluation corpora.
- [`gitingest`](../gitingest/) / [`repomix`](../repomix/) /
  [`code2prompt`](../code2prompt/) — same "input → LLM-ingestible
  text" stance but for source trees, not web pages.

## 7. Hello-world

```bash
# One URL → clean Markdown on stdout
trafilatura -u https://example.com/article \
  --output-format markdown

# A whole site via sitemap → JSONL with metadata
trafilatura --sitemap https://example.com/sitemap.xml --list \
  | xargs -n1 -P 8 trafilatura --output-format json \
  > corpus.jsonl

# Raw HTML on stdin → plain text on stdout
curl -s https://example.com | trafilatura
```

```python
# Library use
import trafilatura
html = trafilatura.fetch_url("https://example.com/article")
text = trafilatura.extract(
    html,
    output_format="markdown",
    with_metadata=True,
    target_language="en",
)
print(text)
```

## 8. Killer feature, weakness, when to choose

- **Killer feature**: the cleanest "URL → Markdown + metadata"
  one-liner in the catalog, with a focused crawler and language
  filter built in, no headless browser required, fast enough to
  build million-page corpora on a single laptop.
- **Weakness**: zero JavaScript execution — silent on SPAs;
  metadata-only structured extraction (no schema-driven field
  extraction); no MCP server.
- **When to choose**: HTML-heavy LLM data prep where the inputs
  are static or server-rendered, you want shell-pipeable speed,
  and a fat browser dep is a non-starter. Promote to
  [`crawl4ai`](../crawl4ai/) the moment you hit a JS-rendered site.

## 9. Repo health (snapshot)

- Active since 2019, steady release cadence on `master`; v2.0.0
  shipped 2024-12 with a major API cleanup and Python ≥ 3.9
  baseline.
- Used as the extraction backbone in several published web-corpus
  projects (CommonCrawl-derived datasets, OSCAR, academic
  digital-humanities pipelines).
- Stable public surface: `fetch_url`, `extract`, `extract_metadata`,
  `bare_extraction`, the `trafilatura` shell front-end. Pin
  `trafilatura>=2,<3` in production.
