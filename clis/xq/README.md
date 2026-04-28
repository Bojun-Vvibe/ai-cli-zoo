# xq

- **Repo:** https://github.com/sibprogrammer/xq
- **Version:** v1.4.0 (latest stable, 2026-02)
- **License:** MIT ([LICENSE](https://github.com/sibprogrammer/xq/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install xq` · `go install github.com/sibprogrammer/xq@latest` · `curl -sSL https://bit.ly/install-xq | sudo bash` (official one-liner) · static release binaries on the GitHub release page for Linux / macOS / Windows / FreeBSD / AIX-PPC64 (amd64 + arm64)

## What it does

`xq` is a `jq`-shaped command-line processor for **XML and HTML**. Pipe a
document into it (or hand it a file path) and it pretty-prints the tree
with syntax highlighting by default, or applies an XPath / CSS-selector
expression to extract just the nodes you want, with optional conversion
to JSON so the result drops straight into a `jq` pipeline. The point is
the same shape as `jq` — Unix filter, stdin-in / stdout-out, no glue
code, no Python `lxml + bs4` script — for the half of the world's data
that is still XML or HTML soup instead of JSON.

## Minimal usage example

```bash
# 1. Pretty-print + colorise an XML file (the "no flag" default)
$ xq pom.xml
# expected: the same XML but indented, attributes aligned, element names
# in cyan, attribute names in yellow, string values in green; long files
# are paged through $PAGER automatically

# 2. Extract every <artifactId> with an XPath query
$ xq -x '//artifactId/text()' pom.xml
# expected: one artifactId per line, e.g.
# my-app
# spring-boot-starter-web
# junit-jupiter

# 3. Scrape an HTML page with a CSS selector + extract one attribute
$ curl -s https://news.ycombinator.com | xq -q 'span.titleline > a' -a href
# expected: one URL per line — every story link on the HN front page;
# combine with `head -n 5` to get the top 5

# 4. Convert an XML config into JSON so jq can take over
$ xq -j config.xml | jq '.config.database.host'
# expected: the host string from the XML, now reachable through a jq path

# 5. Reformat a directory of XML files in place (added in v1.4.0)
$ xq -i src/main/resources/*.xml
# expected: every file rewritten with consistent 2-space indent;
# no stdout output unless a file failed to parse
```

## When to pick it / when not to

Pick `xq` when the data you are wrestling is **XML or HTML, not JSON**
— Maven / Gradle `pom.xml`, Spring config, SOAP responses, RSS / Atom
feeds, OPML, SVG, Android `AndroidManifest.xml`, iTunes Library, OFX
bank exports, scraped web pages — and you want the `jq` reflex (`curl |
xq -q '...' -a href`) instead of writing a 30-line Python script.
Pairs naturally with [`htmlq`](../htmlq/) (htmlq is HTML-only and CSS3
only; xq covers both XML and HTML and adds XPath, so reach for `htmlq`
when you want a smaller binary and only need CSS selectors, reach for
`xq` when the page is XML-flavoured or you need XPath functions like
`count()` / `string-length()`), with [`jq`](../jq/) /
[`jaq`](../jaq/) downstream when you take the `-j` JSON exit, with
[`yq`](../yq/) for the YAML side of the same problem space, with
[`gron`](../gron/) when you want the document flattened to grep-able
paths instead of queried.

Skip it for JSON — `jq` / `jaq` are faster and the canonical tool.
Skip it for client-side-rendered HTML — `xq` does not execute
JavaScript, so a SPA needs `chromium-headless --dump-dom` or Playwright
first. Skip it for transforming XML into a different XML shape — that
is XSLT's job (use `xsltproc` / Saxon); `xq` is read-only on its
input. Skip it inside hot loops over millions of small documents — Go
startup cost dominates; for that scale a native parser (`lxml`,
`html5ever`) inside a long-lived process is the right answer.
