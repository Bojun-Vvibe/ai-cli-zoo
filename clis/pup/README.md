# pup

- **Repo:** https://github.com/ericchiang/pup
- **Version:** v0.4.0 (latest tagged release, 2017-11; project quiet but
  binaries still ship cleanly)
- **License:** MIT ([LICENSE](https://github.com/ericchiang/pup/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install pup` · `go install github.com/ericchiang/pup@latest`
  · prebuilt binaries on the GitHub release page

## What it does

`pup` is the Go-flavoured peer of [`htmlq`](../htmlq/): an HTML filter
shaped like `jq`, taking HTML on stdin, applying a CSS-style selector
expression, and writing the matched fragments back to stdout. The
selector grammar is full CSS3 with a few jQuery-style extensions:
`div.product > a.title`, `a[href^="https://"]`,
`tr:nth-child(odd) td:last-of-type`, `:contains("Sign in")`,
`:not([disabled])`, attribute substring matchers, descendant /
child / sibling combinators. Unlike `htmlq`, `pup` chains *display
functions* on the right-hand side of the selector pipeline:
`pup 'a.story json{}'` emits the matched anchors as a JSON array (id,
class, attributes, text, children) ready to feed [`jq`](../jq/);
`pup 'a.story attr{href}'` returns one href per line;
`pup 'article text{}'` strips tags and emits visible text;
`pup 'main slice{0,3}'` keeps just the first three matches. The HTML5
parser is the standard library's `golang.org/x/net/html`, so it
tolerates unclosed `<li>`, missing `</p>`, weird character encoding,
and `<script>` blocks containing `</`-escapes the way browsers do.
The whole tool is one ~5 MB static Go binary with zero runtime
dependencies — no Python, no Node, no `lxml` to apt-install in CI.

## When to pick it / when not to

Pick `pup` when you have HTML coming out of `curl` and want either
(a) typed JSON to feed [`jq`](../jq/) or [`jaq`](../jaq/), or (b) a
single attribute / text payload to feed the next pipeline stage —
link harvesting, title scraping, table extraction, "give me every
`<pre><code>` block on this page". `curl -s URL | pup '.product
json{}' | jq '.[].children[0].text'` is a one-liner where the Python
`requests` + `bs4` equivalent is a script. The `json{}` display
function is the killer differentiator vs. [`htmlq`](../htmlq/): you
get the parsed DOM as JSON, and the rest of the standard Unix JSON
pipeline takes over. Pair with [`strip-tags`](../strip-tags/) when
the goal is HTML→clean-text for an LLM prompt (pup narrows to the
right region, strip-tags drops chrome from what remains), with
[`htmlq`](../htmlq/) when you want the matched HTML back as HTML
rather than as JSON, and with [`xq`](../xq/) for the same trick on
XML / SVG. Because it is a deterministic offline parser with no JS
execution, it is also the right substrate for repeatable scrape
jobs in CI where headless-browser flakiness is unacceptable.

Skip `pup` for pages whose content is rendered by client-side
JavaScript and not present in the raw HTML response — that is a
Playwright / Puppeteer / `chromedp` / `chromium-headless --dump-dom`
problem, full stop. Skip it for "scrape a hundred pages politely
with retries, rate limiting, robots.txt respect, pagination" — pup
is the parsing layer, not a crawler; combine with `wget --mirror`,
`scrapy`, or [`crawl4ai`](../crawl4ai/) for that. Skip it when you
need to *modify* HTML (re-attribute every link, rewrite every
`<img src>` to a CDN, transform table structure) — you want a real
DOM toolkit (Python `lxml`, Node `cheerio`, Rust `scraper`) where
you can walk the tree and mutate nodes. Note also that the upstream
project has been quiet since 2017; it works because Go's
`golang.org/x/net/html` parser has stayed stable and the selector
grammar was complete on day one, but treat it as a battle-tested
classic rather than an actively-developed roadmap.
[`htmlq`](../htmlq/) is the closest peer if you prefer Rust + a
plain HTML output by default.

## Example invocations

```bash
# Harvest every story link from Hacker News
curl -s https://news.ycombinator.com/ \
  | pup 'span.titleline > a attr{href}'

# Same thing but as structured JSON for downstream jq
curl -s https://news.ycombinator.com/ \
  | pup 'span.titleline > a json{}' \
  | jq -r '.[] | "\(.text)\t\(.attributes.href)"'

# Plain-text body of an article, ready for an LLM prompt
curl -s https://example.com/article \
  | pup 'article text{}'

# First three product cards on a listing page
curl -s https://example.com/shop \
  | pup '.product slice{0,3} json{}'

# Pull JSON-LD blocks and query them with jq
curl -s https://example.com/product/42 \
  | pup 'script[type="application/ld+json"] text{}' \
  | jq '.offers.price'

# Find every link that contains a literal phrase
curl -s https://example.com/ \
  | pup 'a:contains("Documentation") attr{href}'

# Dry-run a CSS selector against a saved page during scraper development
pup 'div.search-result h3 a' < saved-page.html
```
