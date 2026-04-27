# tealdeer

> **A very fast implementation of `tldr` in Rust** — a single
> static binary that fetches and renders the community-maintained
> [tldr-pages](https://github.com/tldr-pages/tldr) cheatsheets
> (one-page, example-driven summaries of common command-line
> tools) with sub-100 ms cold-start lookup, an offline cache,
> shell completion, and ANSI-colored output. Pinned to **v1.8.1**
> (commit `b3cd7b1c216656ea3416a86104363589157477dc`,
> dual-licensed
> [LICENSE-APACHE](https://github.com/tealdeer-rs/tealdeer/blob/main/LICENSE-APACHE)
> /
> [LICENSE-MIT](https://github.com/tealdeer-rs/tealdeer/blob/main/LICENSE-MIT)).

Source: <https://github.com/tealdeer-rs/tealdeer>

## TL;DR

`tldr <command>` prints the half-page "what flag do I want?"
cheatsheet for `tar`, `find`, `ssh`, `git rebase`, `ffmpeg`,
`docker run`, and ~5 000 other commands — five concrete
invocations with one-line annotations, instead of the 40-page
`man tar` you skim past every time. The official `tldr` Node
client is fine; `tealdeer` is the Rust port that starts in
~30 ms instead of ~700 ms, runs offline by default after the
first cache fetch, and ships as a single static binary with no
runtime dependency.

## Install

```bash
# Homebrew (macOS / Linux)
brew install tealdeer

# Cargo (any OS with a Rust toolchain)
cargo install tealdeer

# Linux package managers
# Arch:    pacman -S tealdeer
# Debian:  apt install tealdeer       (trixie+)
# Fedora:  dnf install tealdeer
# Nix:     nix-env -iA nixpkgs.tealdeer

# from a release archive (Linux x86_64)
curl -L https://github.com/tealdeer-rs/tealdeer/releases/download/v1.8.1/tealdeer-linux-x86_64-musl \
  -o /usr/local/bin/tldr
chmod +x /usr/local/bin/tldr

# first run downloads the page cache (~5 MB, ~5 000 pages)
tldr --update

# verify
tldr --version    # tealdeer 1.8.1
```

The binary installs as `tldr`; both `tldr` and `tealdeer` work
when invoked. Set up auto-update with `auto_update = true` in
`~/.config/tealdeer/config.toml` so the cache refreshes
periodically without a manual `--update`.

## License

Dual-licensed under Apache-2.0 OR MIT — see
[LICENSE-APACHE](https://github.com/tealdeer-rs/tealdeer/blob/main/LICENSE-APACHE)
and
[LICENSE-MIT](https://github.com/tealdeer-rs/tealdeer/blob/main/LICENSE-MIT).
Pick whichever license fits your downstream; the standard Rust
ecosystem dual-license. The page content itself is CC-BY-4.0
(from the upstream `tldr-pages` project).

## One Concrete Example

```bash
# 1. the daily-driver use: "what's the flag I want?"
tldr tar
# →  - Extract a (compressed) archive into the current directory:
#       tar xf path/to/file.tar[.gz|.bz2|.xz]
#    - Create a gzip-compressed archive...
#    [5–8 examples, one line each]

# 2. operating-system specific pages
tldr -p linux  ip          # Linux ip(8)
tldr -p osx    pbcopy      # macOS-only command
tldr -p common find        # the cross-platform page

# 3. multi-word command (`git rebase`, `docker compose`, ...)
tldr git rebase
tldr docker compose

# 4. refresh the local page cache (run weekly, or set auto_update)
tldr --update

# 5. list every available page on the current platform
tldr --list

# 6. pick a different language
tldr --language zh tar
tldr --language fr ssh

# 7. seed pages without internet (download the tarball offline,
#    point tealdeer at it)
export TEALDEER_CACHE_DIR=$HOME/.cache/tealdeer
tar -xzf tldr-pages.tar.gz -C "$TEALDEER_CACHE_DIR/tldr-pages/"

# 8. pipe into a pager / search
tldr ffmpeg | grep -A1 'audio'

# 9. run inside a container build (no cache, no network call)
RUN cargo install tealdeer && tldr --update && tldr --list > /pages.txt
```

## Niche It Fills

**`man` pages are reference material — exhaustive, alphabetical,
written for someone implementing the tool.** `tldr` pages are
the inverse: example-first, task-shaped, written for someone
*using* the tool five minutes after first hearing of it. The
upstream `tldr-pages` project owns the *content*; `tealdeer` is
the Rust *client* — it solves the "the official Node client takes
700 ms to print 30 lines" problem and adds offline-first behavior,
so the cheatsheet is reachable on a plane or behind a firewall.

For agent / LLM workflows where a model needs canonical
example-driven syntax for a binary it doesn't recognize,
`tldr <cmd>` is a deterministic, community-curated, license-clean
fallback before scraping a man page or hitting the web.

## Vs Already Cataloged

- **Vs [`bat`](../bat/):** complementary — `bat` is the
  syntax-highlighted `cat` / pager; tealdeer pipes into it
  cleanly (`tldr tar | bat -l help`). Use `bat` to read source
  / configs / man-pages-with-color, tealdeer to read the
  example-first summary.
- **Vs `man`:** `man tar` is exhaustive and authoritative —
  every flag, every standard, every history note. `tldr tar`
  shows the five invocations 95% of users actually want, in
  about 20 lines. Reach for tldr first; reach for man when
  tldr's examples don't cover your case.
- **Vs the official `tldr` Node client (`npm install -g tldr`):**
  same page corpus, same protocol, same output format —
  tealdeer is just faster (no Node startup), smaller (one static
  binary, no `node_modules`), and offline-first by default. If
  you already have Node and don't care about the 700 ms
  startup, the official client is fine; if you don't want a
  Node toolchain in your install footprint, tealdeer is the
  zero-dependency drop-in.
- **Vs `cheat`/`navi`:** `cheat` and `navi` let you write *your
  own* cheatsheets (and navi adds an `fzf` picker for filling in
  variables). tealdeer ships the *community* cheatsheets only
  — no personal sheets. The two are complementary: tealdeer for
  the canonical commands you're learning, navi for the
  team-specific incantations no one else has documented.

## Caveats

- **The cache must be populated before first use.** `tldr foo` on
  a fresh install with no cache returns "Page not found" until
  you run `tldr --update` once. Set `auto_update = true` in
  `~/.config/tealdeer/config.toml` to remove this footgun.
- **Coverage is a long tail.** ~5 000 commands is a lot, but
  niche or proprietary tools (`pulumi preview`, vendor CLIs)
  may not have a page. Contribute upstream at
  [`tldr-pages/tldr`](https://github.com/tldr-pages/tldr) when
  you hit a gap.
- **Pages are example-shaped, not reference-shaped.** A flag
  that exists but isn't part of the "five most useful
  invocations" will not appear. For exhaustive flag listings,
  fall back to `man` or `--help`.
- **Quality varies by language and platform.** English pages on
  the `common` and `linux` platforms are the most complete;
  translated pages may lag a release behind, and platform
  overlays (`osx`, `windows`, `freebsd`, `android`) cover only
  the platform-specific commands.
- **The `tldr` binary name collides with the official Node
  client.** If you install both, `which tldr` decides who wins
  based on `$PATH` order. Pick one.
