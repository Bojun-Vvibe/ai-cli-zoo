# tldr

> **Community-maintained collaborative cheatsheets for ~5,000
> command-line tools — short, example-first man-page replacements
> served as plain Markdown from a CDN, with a dozen interchangeable
> client implementations (Rust, Go, Python, Node, C, …) so any box
> with internet (or a one-time `--update` cache) gets `tldr tar`,
> `tldr ffmpeg`, `tldr git rebase` in one screen of copy-paste-ready
> examples.** Pinned to client-spec **v2.3** (2025-03),
> [LICENSE](https://github.com/tldr-pages/tldr/blob/main/LICENSE.md),
> CC-BY-4.0 (pages) / per-client license for binaries (most MIT).

Source: <https://github.com/tldr-pages/tldr>

## TL;DR

`tldr` is the "I know there is a flag for this, what is it" tool.
Where `man tar` is 1,200 lines of POSIX prose organised by
section, `tldr tar` is *seven* one-line examples (`tar -xf
archive.tar`, `tar -czf out.tar.gz dir/`, `tar -tzf
archive.tar.gz | head`, …) with a one-sentence description per
example. Pages live in one big git repo (`tldr-pages/tldr`) as
plain Markdown, are translated into ~30 languages, and are
syndicated to a CDN-hosted zip (`tldr.sh/assets/tldr.zip`) that
clients fetch and cache locally. *Any* of the dozen clients
(`tealdeer` is the fast Rust one already in this catalog;
`tldr-c-client`, `tldr` Node, `tldr-python-client`, `tldr++`,
`tldrl`, the official `tldr` Node client, …) reads the same
Markdown — pick the client that matches your runtime constraints
and the page database is identical. Authorship is by PR against
`pages/<platform>/<command>.md`; the spec lives at
`tldr-pages/tldr/contributing-guides/style-guide.md`.

## Install

The "tldr" entry in this catalog is the **upstream page database +
spec** — to actually run it on the command line, install one
spec-conforming client. Fastest options:

```bash
# tealdeer — already cataloged separately, Rust, fastest cold start
brew install tealdeer && tldr --update

# Node official client (cross-platform, slowest of the bunch)
npm install -g tldr

# C client (smallest install footprint, good for containers)
brew install tldr      # brew formula 'tldr' is the C client on macOS
sudo apt-get install tldr   # Debian/Ubuntu also ships the C client

# Python client
pip install tldr

# Then sync the page cache (all clients share the same upstream zip)
tldr --update
```

## License

- **Pages (Markdown content):** CC-BY-4.0 — see
  [LICENSE.md](https://github.com/tldr-pages/tldr/blob/main/LICENSE.md).
  Copy / redistribute / adapt freely with attribution to the
  tldr-pages team.
- **Clients (binaries):** per-client; the official Node client is
  MIT, `tealdeer` (Rust) is MIT/Apache-2.0 dual, the C client is
  MIT, the Python client is MIT.
- **Contributor License:** new pages are accepted under the same
  CC-BY-4.0 by the standard PR signoff in the contributing guide.

## One Concrete Example

```bash
# 1. one-time cache bootstrap (and weekly thereafter)
tldr --update

# 2. the daily use — what's the flag for...
tldr ffmpeg            # 8 examples covering convert / extract / concat
tldr git rebase        # interactive, onto, abort
tldr ssh-keygen        # ed25519, change passphrase, fingerprint
tldr tar               # the one you forget every time
tldr find              # the one you really forget every time

# 3. platform-scoped pages (macOS-only flags vs Linux-only flags)
tldr -p osx pbcopy
tldr -p linux journalctl
tldr -p common rsync   # cross-platform examples only

# 4. shell function: fall through to man if no tldr page exists
help() {
  tldr "$@" 2>/dev/null || man "$@"
}

# 5. contribute a page (the actual workflow)
git clone https://github.com/tldr-pages/tldr
cd tldr
cp pages/common/_template.md pages/common/mytool.md
$EDITOR pages/common/mytool.md   # 5–8 examples, follow style-guide.md
# open PR; CI lints with `tldr-lint`
```

## Why This, Not Another

1. **Examples-first beats prose-first for 90% of "what's the flag"
   questions.** A `man` page is exhaustive reference; a `tldr`
   page is the five things people actually do. For a tool you
   use weekly, `tldr` is faster than scanning man; for a tool
   you've never touched, `tldr` is the on-ramp before man.
2. **Client-portable.** The Markdown spec is locked
   (`tldr-pages/tldr/contributing-guides/style-guide.md`,
   client spec at v2.3) so the same page renders identically in
   `tealdeer`, the Node client, the C client, the Python client,
   and the half-dozen community clients. You can swap clients
   for runtime / startup-time / installer reasons without losing
   the cache or your muscle memory.
3. **Offline after first sync.** The cache is a ~5 MB local zip;
   `tldr <cmd>` is filesystem-fast after `tldr --update`. Works
   on airgapped boxes if you scp the cache once. Compare against
   `cheat.sh` (HTTP per query, requires connectivity) or `man -k`
   (no examples, just synopses).

For an LLM-CLI workflow, `tldr <cmd>` is a near-zero-token
preamble before asking a model "build me an `ffmpeg` invocation
that…" — it primes the model with the canonical flag names so
it doesn't hallucinate `--output-file` when the real flag is `-o`.
Pair with [`mods`](../mods/) / [`aichat`](../aichat/):
`tldr ffmpeg | mods "now adapt example 3 to also normalise audio"`.

## Vs Already Cataloged

- **Vs [`tealdeer`](../tealdeer/):** `tealdeer` is *a client* for
  this exact page database. This entry is the **upstream
  database + spec + contribution path**. Use `tealdeer` to
  read pages on a workstation; treat the `tldr-pages/tldr` repo
  as the place to *fix* a wrong example or add a missing tool.
- **Vs [`navi`](../navi/):** `navi` is interactive cheatsheets
  with `<param>` placeholders you fill in via fzf — closer to
  a snippet manager. `tldr` is read-only, copy-paste examples.
  Use `tldr` to *learn* the flags, `navi` to *parameterise* a
  recipe you run weekly.
- **Vs [`cheat`](../cheat/):** `cheat` is your *personal*
  cheatsheets (private notes per command), `tldr` is the
  *community* cheatsheets (shared, reviewed, translated).
  Most users want both: `cheat` for the team-specific
  invocations (`cheat deploy-staging`), `tldr` for the
  upstream-tool flag reminders.
- **Vs `man`:** `tldr` does not replace `man` — it precedes it.
  `tldr <cmd>` for the 5 things you do; `man <cmd>` for the
  obscure flag the examples don't cover.

## Caveats

- **Pages are intentionally shallow.** A `tldr` page caps at
  ~8 examples by style-guide rule — corner cases, every flag,
  every interaction live in `man`. If you need exhaustive
  reference, this is not it.
- **You install a client, not "tldr" itself.** First-time users
  conflate the page database with the binary; the question
  "which `tldr` should I install" has a dozen answers. The
  catalog's recommended default is
  [`tealdeer`](../tealdeer/) — fastest cold start, Rust,
  zero-config; the C client (`brew install tldr` on macOS,
  `apt install tldr` on Debian) is the smallest install if
  Rust is unavailable.
- **Cache staleness is on you.** `tldr --update` is not
  automatic in most clients. A weekly `cron` / `launchd` /
  shell-startup hook is the usual fix. Stale cache = stale
  examples (a flag may have been deprecated upstream).
- **CC-BY-4.0 attribution required for redistributing the
  pages.** If you scrape the page set into your own product
  (LLM training data, a docs site, an internal wiki), you owe
  attribution to the tldr-pages team and a link to the source.
  Quoting one example in a Slack reply is fair use; bulk
  republishing is not.
- **Page quality varies by tool.** Hot tools (`git`, `ffmpeg`,
  `docker`) have hand-curated pages; long-tail tools may have
  a single thin example or none. Fix it with a PR — the
  contribution loop is the actual product.
