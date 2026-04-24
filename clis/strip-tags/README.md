# strip-tags

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/strip-tags>
> Binary name: `strip-tags`

`strip-tags` is the **HTML→text minifier** of the catalog. It takes
HTML on stdin (or files), parses it with `bs4`, throws away the
chrome — `<script>`, `<style>`, navigation menus, footers, inline
attributes you do not care about — and emits clean plain text or
minified HTML suitable for piping into an LLM CLI.

The slot it fills is "I just `curl`'d a webpage and I want to ask a
model about its content without burning 40k tokens on `<svg>` paths
and tracking-pixel `<script>` tags". Where [`marker`](../marker/)
solves that problem for *PDF / EPUB / DOCX*, and where
[`files-to-prompt`](../files-to-prompt/) solves it for *source
trees*, `strip-tags` solves it for *the open web*. The three are
complementary front-ends to the same downstream LLM CLIs.

The canonical pipeline:

```sh
curl -s https://example.com/article | strip-tags | llm 'summarize this'
curl -s https://example.com/article | strip-tags article | llm '...'   # only <article>
curl -s https://example.com/article | strip-tags --minify | llm '...'  # keep tags, drop chrome
```

The first form is the lossy-but-cheap default; the second is the
"tell me what the main content selector is" form for sites with
predictable structure; the third keeps HTML semantics when the
downstream model genuinely benefits from headings/lists/tables.

## 1. Install footprint

- Pure Python, ~10 KB CLI plus `beautifulsoup4` and `lxml`. ~3 MB
  installed.
- `pipx install strip-tags` (recommended), `uv tool install
  strip-tags`, or `pip install strip-tags` inside a venv.
- Zero config. No API key, no network at runtime — `strip-tags`
  reads stdin / files and writes stdout.
- Works on any HTML that `lxml` accepts, which in practice means
  every real-world web page including malformed markup.

## 2. License

Apache-2.0.

## 3. Models supported

**None.** `strip-tags` is not a model client. Like
[`files-to-prompt`](../files-to-prompt/),
[`symbex`](../symbex/), [`ttok`](../ttok/),
[`repomix`](../repomix/), and [`marker`](../marker/) (in its core
mode), it is a context shaper. It produces text that *some other*
catalog CLI consumes.

The intended composition is `curl ... | strip-tags | llm`,
`curl ... | strip-tags | mods`, `curl ... | strip-tags | ttok` (to
check token cost first), or `curl ... | strip-tags | files-to-prompt
- ...` (when you want it folded into a larger prompt envelope).

## 4. MCP support

None. `strip-tags` is a Unix filter. For an MCP-style "fetch and
strip a URL" tool, write a one-line shell wrapper and expose it
from a custom MCP server.

## 5. Sub-agent model

None. One invocation reads input, parses, prunes, prints, exits.
There is no loop, no agent, no LLM call.

## 6. Telemetry stance

**Off, with no opt-in.** No network calls of any kind — `strip-tags`
does not even fetch URLs itself, that is your `curl`'s job. The
binary parses HTML in memory and writes to stdout. Drop-in safe in
air-gapped CI and pre-commit hooks.

## 7. Prompt-cache strategy

Not applicable — `strip-tags` does not call any model. The output is
deterministic for a given input + flag set, which means the
*downstream* LLM CLI can cache on it cleanly (Anthropic prefix
caching, Gemini implicit caching).

## 8. Hot keybinds

No TUI, no REPL. The flag set:

- `cat page.html | strip-tags` — strip everything, plain text out.
- `cat page.html | strip-tags article` — keep only the contents of
  `<article>` selectors (CSS-selector syntax via `bs4.select`).
- `cat page.html | strip-tags 'main, .content'` — multiple
  selectors, comma-separated.
- `cat page.html | strip-tags --minify` — keep HTML structure
  (`<h1>`, `<p>`, `<ul>`, `<table>`) but drop inline attributes,
  `<script>`, `<style>`, `<svg>`, comments. Useful when the model
  benefits from structure (table summarization, document outline
  generation).
- `cat page.html | strip-tags -t script -t style -t nav -t footer
  -t aside` — explicit tag denylist, additive to the defaults.
- `strip-tags page1.html page2.html` — multiple file inputs in
  sequence, joined into one stream.
- `cat page.html | strip-tags --keep-tags table` — strip everything
  *except* `<table>`, useful for "give me only the data tables on
  this page".

## 9. Killer feature, weakness, when to choose

**Killer feature.** Unblocks every "ask the model about a webpage"
workflow without you writing a `bs4` script. The 30-second loop
"`curl URL | strip-tags | llm 'summarize'`" is the entire reason
this tool exists, and it is one line. For sites where the main
content lives under a known selector (`article`, `main`,
`.post-body`), the targeted form drops token cost by another order
of magnitude on top of the default strip — and the savings are
real because most modern web pages are 90%+ chrome by byte.

**Weakness.** Three:

1. **No JavaScript rendering.** `strip-tags` parses static HTML.
   Single-page apps (React / Vue / SvelteKit pages whose content
   is JS-rendered) will yield empty or near-empty output because
   the HTML you `curl`'d does not contain the content. For those,
   pre-render with `playwright`, `puppeteer`, `chromium-headless`,
   `monolith`, or [`marker`](../marker/) (PDF), then pipe the
   result in.
2. **No URL fetching.** `strip-tags` does not have a `--url` flag.
   You always pair it with `curl`, `wget`, `http`, or your fetcher
   of choice. That keeps the tool small but means you handle
   redirects, cookies, and auth headers yourself.
3. **Selector syntax is `bs4.select`, not full XPath.** That is
   CSS selectors only — fine for `article`, `main`, `.foo`, `#bar`,
   `div > p`, but no `//div[contains(@class,"foo")]/text()`. For
   XPath, reach for `xmllint --xpath` or a Python script.

**When to choose.**

- **You want to ask a model about a single article / docs page /
  blog post.** `curl URL | strip-tags | llm 'TL;DR'` is the entire
  workflow.
- **You are building a quick research pipeline** that fetches a
  list of URLs and summarizes each. Pair `strip-tags` with `xargs`
  and [`mods`](../mods/) — the per-page token cost makes this
  affordable at hundreds-of-URL scale where naive HTML would not.
- **You need deterministic, no-network HTML cleaning** in CI (e.g.
  cleaning fixture HTML before snapshot-comparing model output).
  `strip-tags` is pure-function on its input.
- **You want to keep table / heading structure** for the model to
  reason over but drop everything else. `--minify --keep-tags
  table,h1,h2,h3` is the form.

**When not to choose.**

- **The page is a SPA whose content is JavaScript-rendered.** The
  HTML `curl` returns is mostly empty. Reach for a headless-browser
  pre-renderer first; `strip-tags` is the *second* stage.
- **You are processing a PDF / EPUB / DOCX, not HTML.** That is
  [`marker`](../marker/)'s job.
- **You need site-specific extraction with field-level precision**
  (product price, author byline, publish date). That is a
  scraping framework (`scrapy`, `parsel`, `playwright` with
  selectors); `strip-tags` is for "give me the gist, not the
  schema".
- **You want the model to *navigate* the web itself.** Reach for a
  browsing agent — [`OpenHands`](../openhands/) browser agent,
  [`gptme`](../gptme/) browser tool, or a [`claude-code`](../claude-code/)
  setup with a fetch tool. `strip-tags` is for "I already have the
  HTML, clean it".
