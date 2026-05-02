# muffet

- **Repo:** https://github.com/raviqqe/muffet
- **Version:** v2.11.3 (latest stable, April 2026)
- **License:** MIT ([LICENSE](https://github.com/raviqqe/muffet/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install muffet` · `go install github.com/raviqqe/muffet/v2@v2.11.3` · static binaries on the GitHub release page · binary name is `muffet`

## What it does

`muffet` is a fast, parallel, single-binary website link checker.
Point it at a URL and it crawls every internal page reachable from
that origin, extracts every `href`, `src`, `srcset`, `action`, and
fragment (`#anchor`) reference, and concurrently HEADs / GETs each
external target — reporting any link that 4xx's, 5xx's, times out,
loops, or points at an `id` that doesn't exist on the destination
page. The default concurrency is high (hundreds of in-flight
requests), so a 1 000-page documentation site finishes in seconds
instead of the minutes a sequential crawler would take.

What sets it apart from the usual suspects (`linkchecker`, `wget
--spider`, `lychee`) is the combination of **fragment validation**,
**HTTP/2 + connection reuse**, and a one-line install. Fragment
checking — verifying that `https://example.com/docs/api#auth-flow`
actually has an element with `id="auth-flow"` — is the failure mode
that breaks documentation sites quietly: the page exists, so a 200
liar like `wget --spider` is happy; meanwhile every "Jump to the
auth section" link in your blog post is dead. `muffet` parses the
target HTML and confirms the anchor is real.

The CLI surface is deliberately small: one positional URL, a
handful of flags for concurrency / timeout / depth, regex
allow/deny lists for which URLs to follow vs. which to verify but
not crawl, an `--ignore-fragments` escape hatch for sites that
generate IDs at runtime, custom request headers, and a `--json`
output mode for downstream tooling.

## When to pick it / when not to

Pick `muffet` whenever you ship a **documentation site, a static
blog, or a published OpenAPI / catalog**. The CI pattern is one
line — `muffet --buffer-size 8192 --max-connections 100 https://
docs.example.com` — and it catches the entire class of "link
rotted, anchor renamed, page moved without a redirect" bugs that
otherwise reach the reader. It is also the right tool to verify a
freshly-built static site **before** publishing: build to `dist/`,
serve it locally on `:8000`, run `muffet http://localhost:8000`,
fail the deploy on any non-zero exit.

Pick it for **catalog repos** like this one: a `clis/` directory
where every entry has a "Repo:" GitHub link and a license-file
reference is exactly the shape `muffet` was built to verify, and
the fragment check catches "I deep-linked to a section that got
renamed in the upstream README" the moment it happens.

Skip it when:

- You need to **scrape data**, not verify links — that's
  [`colly`](https://github.com/gocolly/colly),
  [`scrapy`](https://scrapy.org/), or
  [`playwright`](https://playwright.dev/), not a link checker.
- You need to **render JavaScript** before extracting links — `muffet`
  reads the server-rendered HTML and does not execute scripts. If
  your site is a client-rendered SPA whose links only exist after
  hydration, run [`playwright`](https://playwright.dev/) or
  [`puppeteer`](https://pptr.dev/) and feed the rendered DOM to a
  link checker, or use `--exclude` to skip those pages entirely.
- You want to check links inside Markdown / source files **without
  rendering them** — that's [`lychee`](https://github.com/lycheeverse/lychee),
  which natively parses Markdown, AsciiDoc, and source comments;
  use both (`lychee` for repo files, `muffet` for the rendered
  site) and you have full coverage.
- You need **archived snapshots** of dead links for compliance —
  `muffet` only reports the failure; pair with the
  [Internet Archive's `wayback` CLI](https://github.com/akamhy/waybackpy)
  if you also need a preserved copy.

## Why it matters in an AI-native workflow

LLM agents writing documentation, blog posts, or catalog entries
hallucinate URLs constantly: real-looking GitHub paths that don't
exist, anchor links to sections the agent invented, version
numbers that were never released. The default human review path
("does this link work?") doesn't scale once the agent is producing
dozens of entries a day. `muffet` is the cheap automatic gate:
build the site, run the checker, fail the PR if anything 404s or
points at a missing fragment. The agent sees the failure list,
fixes the broken references, retries — closed loop, no human
needed for the URL-existence step.

The fragment check is the part that matters specifically for
agent-generated docs. Agents love deep-linking ("see the `#auth`
section of the API docs"), and they love inventing plausible
section names. Without fragment validation, a `200 OK` is
indistinguishable from a real working anchor; with it, the
"section was hallucinated" failure is reported the same way a
broken page is. The `--json` output makes this trivially
machine-readable: `{"url": "...", "error": "id #auth-flow not
found"}` is exactly the shape the agent's next-iteration prompt
needs.

## Example invocations

```bash
# Check a published documentation site, default settings.
muffet https://docs.example.com

# CI-shaped invocation: bigger buffer for large pages, capped concurrency,
# explicit timeout, JSON output for the dashboard.
muffet --buffer-size 8192 --max-connections 100 \
       --timeout 20 --format json \
       https://docs.example.com > linkcheck.json

# Verify a locally-built static site before deploy.
python3 -m http.server 8000 --directory ./public &
muffet --max-connections 50 http://localhost:8000

# Skip outbound link checking for a known-noisy host
# (e.g., an API that rate-limits unauthenticated requests).
muffet --exclude '^https://api\.github\.com/' https://docs.example.com

# Check fragments only on internal links (external sites often generate
# IDs at runtime that the server-rendered HTML doesn't expose).
muffet --ignore-fragments '^https://(?!docs\.example\.com).*' \
       https://docs.example.com

# Send a custom Accept-Language header (sites that geo-redirect).
muffet --header "Accept-Language: en-US" https://docs.example.com

# Limit crawl depth to 3 pages from the seed.
muffet --max-redirections 5 --buffer-size 4096 https://docs.example.com
```

## Caveats

- `muffet` does not execute JavaScript. A SPA whose nav menu is
  hydrated client-side will appear to have no internal links;
  use server-side rendering or a pre-render step before checking.
- Aggressive concurrency (the default is high) can trip a target
  site's WAF / rate limiter and produce spurious 429s. For external
  domains, dial `--max-connections-per-host` down to a polite value
  (4–8) before assuming the failures are real.
- The fragment check trusts the rendered HTML's `id=` attributes.
  If your static site generator puts anchor IDs on a wrapper
  element instead of the heading itself, results may surprise you;
  match against the generator's actual output, not the Markdown
  source.
- HTTP basic auth, OAuth-protected pages, and login-walled docs
  cannot be crawled without the right `--header` plumbing, and
  `muffet` cannot perform an interactive login flow. For those,
  serve a build artifact locally and check that.
- Exit code is non-zero on any reported failure, which is what you
  want in CI but worth knowing if you script around it: `muffet …
  || true` if you only want to log, not gate.
