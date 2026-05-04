# xidel

> **Single-binary CLI for extracting and transforming data from
> HTML, XML, and JSON via XPath 3.0, XQuery 3.0, CSS selectors,
> JSONiq, and a built-in pattern-matching template language** —
> downloads pages over HTTP/S directly (cookies, redirects, forms,
> multi-step crawls), parses them with a fault-tolerant HTML
> parser, then runs full XPath 3 / XQuery 3 (not the cut-down
> XPath 1.0 most tools ship) against the resulting tree, and
> emits the result as plain text, JSON, XML, bash variables,
> or templated output. Pinned to **v0.9.8** (released
> 2022-09-25,
> [LICENSE](https://github.com/benibela/xidel/blob/Xidel_0.9.8/LICENSE),
> GPL-3.0).

Source: <https://github.com/benibela/xidel>

## TL;DR

The HTML/XML scraping CLI space splits into (a) one-language
selectors — [`pup`](../pup/) (CSS only), [`htmlq`](../htmlq/)
(CSS only), [`xq`](../xq/) (XPath 1.0 via `xmlstarlet` shape) —
fast and tiny, but you hit the ceiling the moment you need
joins, conditionals, computed values, or JSON-and-HTML in the
same query, and (b) full programming environments — Python +
`lxml` / `parsel` / `BeautifulSoup`, Node + `cheerio`. `xidel`
sits in the middle: it speaks XPath 3.0 / XQuery 3.0 / JSONiq,
which means `for`, `let`, `where`, `order by`, `group by`,
function definitions, regex, math, string manipulation, *and*
HTML / JSON / XML traversal in one expression — all from a
single 5 MB binary with no runtime. The HTTP client is built
in (cookies persist across `--follow` chains, forms POST with
`--post`, login flows compose), so a multi-page crawl + extract
+ JSON-emit pipeline is one command, no Python venv.

## Install

```bash
# macOS (Homebrew)
brew install xidel

# Debian / Ubuntu
sudo apt-get install -y xidel

# Arch (AUR)
yay -S xidel

# Prebuilt static binary (Linux x86_64)
curl -L -o xidel.tar.gz \
  https://github.com/benibela/xidel/releases/download/Xidel_0.9.8/xidel-0.9.8.linux64.tar.gz
tar -xzf xidel.tar.gz && sudo mv xidel /usr/local/bin/

# From source (FreePascal)
git clone https://github.com/benibela/xidel.git && cd xidel
make && sudo make install

# Verify
xidel --version           # Xidel 0.9.8
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/benibela/xidel/blob/Xidel_0.9.8/LICENSE).
Copyleft: redistributing modified `xidel` requires the same
licence; *output* of running `xidel` against your own data is
not encumbered. The runtime is FreePascal; no Python / Node /
JVM dependency.

## Common invocations

```bash
# CSS selector, like pup
xidel https://example.org -e 'css("h1")'

# XPath 3.0 — full language, not 1.0
xidel https://news.example.org -e '//article[.//time/@datetime > "2026-05-01"]/h2/string()'

# XQuery 3.0 with for / let / where / order-by
xidel https://shop.example.org/products -e '
  for $p in //div[@class="product"]
  let $price := number($p//span[@class="price"]/replace(., "[^0-9.]", ""))
  where $price < 50
  order by $price
  return concat($p//h3, " — $", $price)'

# JSON in, JSONiq out — same engine
curl -s https://api.example.org/orders | xidel - -e '
  for $o in $json("orders")[]
  group by $u := $o("user_id")
  return { "user": $u, "total": sum($o("amount")) }'

# Multi-step crawl: login form → fetch → extract
xidel https://site.example.org/login \
  --post 'user=alice&pass=$PASS' \
  --follow '//a[@href="/dashboard"]/@href' \
  -e '//table[@id="invoices"]//tr/td[1]/string()'

# Output as bash variables — eval directly
eval "$(xidel https://example.org -e '
  title := //title/string(),
  desc  := //meta[@name="description"]/@content
' --output-format=bash)"
echo "$title — $desc"

# Templated output (Xidel pattern matcher)
xidel https://example.org/blog -e '
  <html><body>
    <article>
      <h2>{title:=.}</h2>
      <time>{date:=.}</time>
    </article>+
  </body></html>
' --output-format=json-wrapped
```

## Why use it

- **XPath 3.0 / XQuery 3.0, not 1.0.** Real `for`/`let`/`where`,
  `group by`, regex (`matches`, `replace`, `tokenize`), maps,
  arrays, function definitions (`function($x) { ... }`),
  higher-order operators. Most XPath CLIs in the wild are
  XPath 1.0, which can't do half of these.
- **HTML, XML, and JSON in one query language.** `$json("key")`
  inside an XPath expression treats a JSON document as a
  navigable tree alongside HTML — joining a scraped page
  against an API response is one expression.
- **Built-in HTTP with cookies / forms / follow chains.**
  `--follow` walks links by XPath, `--post` submits forms,
  cookies persist by default. No `curl | xidel -` two-step
  for multi-page flows.
- **Single static binary, no runtime.** ~5 MB FreePascal
  binary; ships in apt and Homebrew; no Python venv, no
  npm install, no JVM. Slots into Alpine containers and CI
  jobs with one `apk add xidel` (community repo) or one
  `curl | tar`.

## Vs Already Cataloged

- **Vs [`pup`](../pup/):** `pup` is CSS-selector-only and
  HTML-only — perfect for "give me all `a.product-link`
  hrefs". When you need conditionals, math, joins across
  documents, regex extraction, or JSON-plus-HTML,
  `pup` runs out of language and `xidel` keeps going.
- **Vs [`htmlq`](../htmlq/):** same shape as `pup` (CSS,
  HTML), same ceiling. Pick `htmlq` for tiny one-liners,
  `xidel` when the query has logic.
- **Vs [`xq`](../xq/):** `xq` (the `yq`-family one) wraps
  `jq`-style expressions over XML; great for XML-as-JSON
  workflows. `xidel` stays in XPath/XQuery shape, which is
  the right idiom when the input *is* an XML/HTML document
  with namespaces, mixed content, and ancestor axes.
- **Vs [`jq`](../jq/) / [`gojq`](../gojq/) / [`gron`](../gron/):**
  pure-JSON tools. `xidel` is the choice when the same
  pipeline straddles JSON *and* HTML/XML; for pure JSON
  workloads `jq` is more idiomatic and faster.
- **Vs [`crawl4ai`](../crawl4ai/) / [`firecrawl`](../firecrawl/):**
  those are LLM-aware crawlers that emit clean Markdown for
  RAG. `xidel` is the surgical extractor — you tell it
  exactly which nodes you want and what shape to emit.
  Pair: `xidel` for structured-extraction CI jobs, the
  LLM crawlers for "summarise this site for embedding".

## Caveats

- **GPL-3.0.** Distributing modified `xidel` requires
  re-licensing under GPL-3.0. Calling `xidel` from a
  proprietary script and shipping its *output* is fine.
- **Release cadence is slow.** v0.9.8 (2022-09-25) has been
  the stable line for years; the project is maintained but
  not fast-moving. The XPath/XQuery 3.0 surface is mature
  and stable, so this rarely bites.
- **HTML parser is forgiving but not a full browser.**
  JavaScript-rendered SPAs are invisible — there is no
  headless Chromium. Pair with `chromedp` /
  `playwright-cli` to render first, pipe the rendered HTML
  into `xidel -` to extract.
- **XPath/XQuery 3.0 has a learning curve.** Worth it if
  you scrape often; overkill for a one-off CSS-selector
  job (use `pup` / `htmlq` instead).
- **Self-hosted FreePascal binary.** Less common toolchain
  than Go / Rust; if you build from source, expect to
  install `fpc` and `lazarus` build deps first.
