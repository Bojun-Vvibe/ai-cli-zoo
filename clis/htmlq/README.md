# htmlq

- **Repo:** https://github.com/mgdm/htmlq
- **Version:** v0.4.0 (latest tagged release, 2022-01)
- **License:** MIT ([LICENSE.md](https://github.com/mgdm/htmlq/blob/master/LICENSE.md))
- **Language:** Rust
- **Install:** `brew install htmlq` · `cargo install --locked htmlq` · binary releases on the GitHub release page

## What it does

`htmlq` is `jq` for HTML: a Unix-shaped filter that takes HTML on stdin,
applies a CSS selector you pass on the command line, and writes the
matched fragments back to stdout. Built on the same `html5ever` parser
that powers Servo and Firefox's experimental engine, so it handles real
real-world soup — unclosed `<li>`, dropped `</p>`, character encoding
declared three different ways, `<script>` blocks containing `</`-
escapes, malformed `<!DOCTYPE>` — without choking the way regex-on-HTML
hacks do. The selector grammar is full CSS3: `div.product > a.title`,
`a[href^="https://"]`, `tr:nth-child(odd) td:last-of-type`,
`section#main h2 + p`, attribute substring matchers, descendant
combinators, the lot. Output is configurable: `--text` strips tags and
returns visible text only, `--attribute href` returns just the value of
one attribute (the killer combo for `curl … | htmlq -a href 'a.story'`
to harvest URLs), `--pretty` reflows the matched HTML through an
indenting serializer, and the default returns the matched fragments
verbatim. A handful of small flags cover the rest: `--ignore-whitespace`
collapses runs of whitespace in text mode, `--base URL` resolves
relative `href`/`src` against a base URL so harvested links are usable,
and `--remove-nodes 'selector'` deletes matching elements (clearing out
ad slots, navigation chrome, or `<script>` tags) before further
processing. The whole thing is a single ~3 MB static Rust binary with
zero runtime dependencies.

## When to pick it / when not to

Pick `htmlq` whenever you have HTML coming out of `curl` and want
structured data going into the next pipeline stage — link harvesting,
title scraping, table extraction, "give me every `<pre><code>` block on
this page", removing the cookie banner before piping to a Markdown
converter. The Unix-pipe shape is the value: `curl -s URL | htmlq -a
href 'a.product-card__link' | sort -u` is a one-liner where the Python
`requests` + `bs4` equivalent is a script. It composes naturally with
[`strip-tags`](../strip-tags/) when you want HTML→clean-text for an LLM
prompt (htmlq narrows to the right region, strip-tags drops the rest of
the chrome), with [`jq`](../jq/) / [`jaq`](../jaq/) when the page
embeds JSON-LD inside `<script type="application/ld+json">` (htmlq
extracts the script body, jq queries it), with [`pandoc`](https://pandoc.org)
or [`mdcat`](https://github.com/swsnr/mdcat) for "render this `<article>`
to Markdown / terminal", and with [`xq`](../xq/) for the matching trick
on XML / SVG. Because it is a deterministic offline parser with no JS
execution, it is also the right substrate for repeatable scrape jobs in
CI, where headless-browser flakiness is unacceptable.

Skip it for any page where the content you want is rendered by
client-side JavaScript and not present in the raw HTML response — that
is a Playwright / Puppeteer / `chromedp` / `chromium-headless --dump-dom`
problem, full stop. Skip it for "scrape a hundred pages politely with
retries, rate limiting, robots.txt respect, and pagination" — htmlq is
the parsing layer, not a crawler; combine with `wget --mirror`,
`scrapy`, or `crawlee` for that. Skip it when you need to *modify*
HTML beyond the limited `--remove-nodes` surface (re-attribute every
link, rewrite every `<img src>` to a CDN, transform table structure) —
you want a real DOM toolkit (`pup`, Python `lxml`, Node `cheerio`, Rust
`scraper` library directly) where you can walk the tree and mutate
nodes. And note the project has been quiet since the v0.4.0 tag in
early 2022; it works because the underlying `html5ever` and CSS-
selector crates have stayed stable, but treat it as a battle-tested
classic rather than an actively-developed roadmap. If you need a more
recently-maintained alternative with the same Unix-pipe shape,
[`pup`](https://github.com/ericchiang/pup) (Go, last touched 2024) is
the closest peer.

## Example invocations

```bash
# Harvest every story link from Hacker News
curl -s https://news.ycombinator.com/ \
  | htmlq --attribute href 'span.titleline > a'

# Plain-text body of an article, ready for an LLM prompt
curl -s https://example.com/article \
  | htmlq --text 'article'

# Resolve relative URLs against a base while extracting
curl -s https://docs.example.com/index.html \
  | htmlq --base https://docs.example.com --attribute href 'nav a'

# Strip noise (scripts, nav, ads) then convert what remains to Markdown
curl -s https://example.com/post \
  | htmlq --remove-nodes 'script,style,nav,.ad' 'article' \
  | pandoc -f html -t markdown

# Pull JSON-LD blocks and query them with jq
curl -s https://example.com/product/42 \
  | htmlq --text 'script[type="application/ld+json"]' \
  | jq '.offers.price'

# Extract a table to TSV (very rough; works for simple tables)
curl -s https://example.com/stats.html \
  | htmlq --text 'table#leaderboard tr' \
  | paste - - - -

# Dry-run a CSS selector against a saved page during scraper development
htmlq 'div.search-result h3 a' < saved-page.html

# Pretty-print the matched fragment for visual inspection
curl -s https://example.com/ | htmlq --pretty 'main' | less
```
