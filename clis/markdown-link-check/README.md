# markdown-link-check

> **markdown-link-check** — tcort/markdown-link-check, a CLI that
> extracts every hyperlink from a Markdown file (or stream of
> Markdown files) and resolves each one over HTTP / HTTPS / mailto /
> file scheme to detect dead links *before* the broken docs ship.
> Pinned to **v3.14.2**, ISC — license file:
> [LICENSE.md](https://github.com/tcort/markdown-link-check/blob/master/LICENSE.md).

Source: <https://github.com/tcort/markdown-link-check>

## TL;DR

```bash
npx markdown-link-check README.md
# or
npm install -g markdown-link-check
markdown-link-check ./docs/**/*.md
```

For each Markdown file, the tool walks the AST (via `markdown-link-extractor`),
extracts every `[text](url)`, `<url>`, reference-style `[text][ref]`,
and image link, then issues an HTTP HEAD (falling back to GET when
the server rejects HEAD) against each URL with a configurable
concurrency cap. Output is one line per link with status — `✓ alive`,
`✗ dead`, or `~ skipped` (if the URL matched a config skip pattern).

Exit code is non-zero on any dead link, so a single
`find docs -name '*.md' | xargs -n1 markdown-link-check` step in CI
gates the merge.

## What it actually solves

Documentation rot. Every reference to an upstream blog post, RFC,
SO answer, or sibling-doc heading anchor in a long-lived repo has a
half-life — domains expire, posts get redacted, redirect chains
break, anchors get renamed when the doc author refactors headings.
The "did the docs PR break any links?" check is one of those things
every docs site reinvents (Sphinx has `linkcheck`, Hugo has
`htmltest`, MkDocs has third-party plugins). markdown-link-check is
the **format-agnostic, zero-build, takes-a-Markdown-file-path**
version that drops into any repo with a Node toolchain.

It's also widely deployed as a **GitHub Action** (`gaurav-nelson/github-action-markdown-link-check`
wraps it) which is how most consumers run it — the underlying CLI
is npm-installable and works the same locally.

## Configuration

A `.markdown-link-check.json` in CWD (or supplied via `--config`)
controls behavior:

```json
{
  "ignorePatterns": [
    { "pattern": "^https://example\\.com/private/" },
    { "pattern": "^http://localhost" }
  ],
  "replacementPatterns": [
    { "pattern": "^/", "replacement": "https://my-site.example.com/" }
  ],
  "httpHeaders": [
    {
      "urls": ["https://api.github.com"],
      "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" }
    }
  ],
  "timeout": "20s",
  "retryOn429": true,
  "retryCount": 3,
  "fallbackRetryDelay": "30s",
  "aliveStatusCodes": [200, 206]
}
```

Key knobs:
- `ignorePatterns` — silence intranet / private / known-flaky URLs
- `replacementPatterns` — rewrite root-relative `/foo/bar` links to a
  staging or prod hostname so root-anchored docs check correctly
- `httpHeaders` — pass auth tokens for rate-limited APIs (GitHub) so
  the tool doesn't get throttled to 429s
- `retryOn429` — back off and retry on rate-limit responses (essential
  for repos with many `https://github.com/...` links)
- `aliveStatusCodes` — accept 206 partial-content for big-file URLs

## Why orthogonal to the existing zoo

The zoo has many doc-quality and link-related tools, but the niche
is precise:

- [`lychee`](../lychee/) is the closest neighbor — a Rust link
  checker that handles Markdown *and* HTML *and* plaintext, with
  much higher concurrency and a cache. **lychee wins on speed,
  scope, and cache**; markdown-link-check wins on **the
  GitHub-Action ecosystem**, the per-file invocation model that
  docs CI traditionally uses (lychee is a single multi-file run by
  design), the simpler config surface, and the Node-native install
  for repos that already have a `package.json` (one less toolchain).
  They coexist: lychee for the high-volume, mixed-format link audit
  on a static site; markdown-link-check for the per-PR gate on a
  README-and-docs-folder repo on Node CI.
- [`muffet`](../muffet/) is a parallel HTTP link checker for
  *deployed websites* (it crawls a URL recursively). Different
  verb — markdown-link-check operates on source Markdown files
  *before* deploy; muffet operates on the rendered site *after*.
  Pair both for end-to-end coverage.
- [`htmlq`](../htmlq/) extracts content from HTML — adjacent shape
  but not a checker.
- [`mdformat`](../mdformat/) / [`markdownlint-cli2`](../markdownlint-cli2/)
  enforce *style and structure* of Markdown (heading levels, list
  indent, trailing whitespace) — markdown-link-check is the only
  tool of this family that actually **resolves URLs over the
  network**, the orthogonal axis.
- [`vale`](../vale/) is the prose-style linter (write-good,
  Google style, etc.) — checks the *words*, not the *links*.
- [`mdbook`](../mdbook/) has a `linkcheck` backend, but it's mdBook-
  specific; markdown-link-check works on any Markdown file
  regardless of the static-site generator.

In practice: drop markdown-link-check into a `docs.yml` GitHub
Action with the `gaurav-nelson` wrapper for the per-PR docs gate.
Use lychee for the heavier full-site audit job that runs nightly.

## Pairs with

- [`vale`](../vale/) — prose linting on the same docs (links + style
  in the same CI job)
- [`markdownlint-cli2`](../markdownlint-cli2/) — Markdown structure
  linting (heading hierarchy, list indent)
- [`mdformat`](../mdformat/) — auto-format the same Markdown
- [`lychee`](../lychee/) — heavier full-site link audit on schedule
- [`gh`](../gh/) — pass `GITHUB_TOKEN` via `httpHeaders` so the GitHub
  API URLs in docs don't 429

## Caveats

- Concurrency defaults are conservative (5 in-flight requests) — fine
  for small docs trees, slow for repos with hundreds of files. Bump
  `--retryCount` and the per-host concurrency or switch to lychee for
  large audits.
- HEAD-then-GET fallback is correct but doubles the request count
  against servers that 405 HEAD; cache misses cost.
- Anchor-fragment checks (`https://site/#section`) require the tool
  to fetch and parse the page — supported but slower; some servers
  return 200 for non-existent anchors so false-negatives exist.
- ISC-licensed Node package — needs a Node runtime on the build agent.
  Use lychee if you want a single static binary with no Node dep.
- Project cadence is steady but not daily — pin the version in CI to
  avoid surprise behavior changes.
