# percollate

> **Web pages → clean PDF / EPUB / HTML / Markdown bundles**
> — a Node.js CLI that takes one or many URLs, runs each
> through a headless Chromium + Mozilla Readability pipeline
> to strip nav/ads/sidebars/cookie banners, and emits a single
> typeset document (PDF for print, EPUB for an e-reader, HTML
> for archival, Markdown for note-taking). Pinned to **v4.3.0**
> ([LICENSE](https://github.com/danburzo/percollate/blob/master/LICENSE),
> MIT).
>
> Multi-URL inputs are merged into one cover-paged volume,
> not a stack of separate files — perfect for binding a
> reading list into a flight-mode bundle.

Source: <https://github.com/danburzo/percollate>

## TL;DR

`percollate pdf <url>...` produces a typographically respectable
PDF of an article — real margins, real fonts, real page breaks,
hyphenation, footnoted source URLs, and a generated cover with
title / author / date. `percollate epub <url>...` produces an
EPUB3 with chapter-per-URL navigation, suitable for sideloading
to a Kindle / Kobo / reMarkable. `percollate html` and
`percollate md` give you the cleaned-up content as a single
HTML or Markdown file — the foundation that the PDF/EPUB
renderers build on.

The Readability pass is the same engine Firefox Reader View
uses, so the "what counts as the article" boundary matches
what humans expect on news sites, blogs, docs, and longform.
You can override the article selector per-site, inject custom
CSS, hide elements, change page size / margins / font, and
toggle the cover, table of contents, and per-link footnotes.

## Why it's interesting

The "save this for offline reading" problem is solved
fragmentarily — Pocket, Instapaper, browser reader views,
ad-hoc `wkhtmltopdf` scripts, paywalled "send-to-Kindle"
services. None of them give you **one local, scriptable
binary that emits an actual book-quality artifact** from a
list of URLs. `percollate` does. The output is good enough
to print and bind, good enough to read on an e-ink device
without reformatting, and small enough to commit to a
research notebook repo.

Because it's a normal CLI it composes: `cat reading-list.txt
| xargs percollate epub -o week-42.epub`, or as a cron job
that turns an RSS feed into a daily EPUB delivered to your
e-reader.

## Install

```bash
# Node 18+ required (bundles puppeteer / chromium)
npm install -g percollate

# verify
percollate --version    # 4.3.0
```

## Examples

```bash
# single article → PDF
percollate pdf https://example.com/longform-essay

# multiple URLs → one combined EPUB with cover + TOC
percollate epub \
  https://example.com/post-1 \
  https://example.com/post-2 \
  https://example.com/post-3 \
  -o weekend-reading.epub

# Kindle-friendly page size + sans-serif body
percollate pdf --css 'body { font-family: sans-serif; }' \
  --size A6 \
  -o pocket.pdf https://example.com/post

# strip the cover + TOC, keep just the cleaned articles
percollate pdf --no-cover --no-toc -o plain.pdf https://example.com/post

# pull the cleaned-up Markdown for a personal notes repo
percollate md -o notes/post.md https://example.com/post

# pipe a reading list
xargs -a urls.txt percollate epub -o digest.epub
```

## Use when

- You want offline reading artifacts (PDF for print, EPUB for
  e-reader) that look hand-typeset, not "Save Page As".
- You're building a research/notes workflow and want
  Readability-cleaned Markdown of every URL you cite, stored
  in a repo, regeneratable.
- You curate a weekly digest (newsletter, team reading list,
  flight bundle) and want it emitted as one bound document
  rather than a folder of HTML.
- You need a scriptable, self-hosted alternative to Pocket /
  Instapaper / "send to Kindle" services with no third-party
  account.

Skip `percollate` when the page requires JavaScript-driven
auth or paywalled content you don't have access to (it can't
log in for you), or when you only want to skim — opening the
URL in a browser is faster than spinning up a headless
Chromium render.
