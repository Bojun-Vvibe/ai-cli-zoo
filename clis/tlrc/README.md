# tlrc

> **The official Rust client for `tldr-pages`** —
> looks up community-maintained, example-first man-page
> alternatives in <50 ms with zero network calls in the
> hot path. Pinned to **v1.13.0**
> ([LICENSE](https://github.com/tldr-pages/tlrc/blob/main/LICENSE),
> MIT).

Source: <https://github.com/tldr-pages/tlrc>

## TL;DR

`tlrc` is the third-generation native client for
[`tldr-pages`](https://github.com/tldr-pages/tldr) — the
~15,000-page community wiki of "show me three useful
invocations of `tar` instead of seven screens of `man tar`."
It is the *officially endorsed* client maintained inside
the `tldr-pages` GitHub org itself, replacing the older
[`tealdeer`](../tealdeer/) (still excellent and also in
the zoo) as the upstream-blessed default. The model is
the same: pages are short markdown files (one per command,
plus per-OS overrides) shipped as a versioned cache; the
client downloads the cache once, then every lookup is a
local file read. `tlrc` keeps the cache under
`~/.cache/tlrc/`, refreshes it on a configurable interval
(default: 14 days, with `--update` for an explicit pull),
and renders pages with proper ANSI color, configurable
themes, and per-platform fallthrough (`tlrc -p macos
launchctl` first, then `common`). Compared to
`tealdeer`, `tlrc` ships with stricter cache integrity
(SHA-256-verified zip downloads, lockfile-protected
updates so two parallel `tlrc --update` calls don't
corrupt the cache), a richer config (`~/.config/tlrc/config.toml`
with full theme + platform + cache control), and tighter
upstream sync — when a new `tldr-pages` release lands,
`tlrc`'s default page set is bumped within hours rather
than waiting on a third-party maintainer.

## Install

```bash
# Homebrew (macOS / Linux)
brew install tlrc

# Cargo
cargo install --locked tlrc

# Arch
pacman -S tlrc

# Nix
nix-env -iA nixpkgs.tlrc

# Alpine
apk add tlrc

# from a release binary
curl -LO https://github.com/tldr-pages/tlrc/releases/download/v1.13.0/tlrc-v1.13.0-x86_64-unknown-linux-musl.tar.gz
tar xzf tlrc-v1.13.0-x86_64-unknown-linux-musl.tar.gz
sudo install tlrc /usr/local/bin/

# verify
tlrc --version    # tlrc 1.13.0
```

`tlrc` ships a single binary named `tlrc` plus an alias
target named `tldr` (some packagers install both, others
just `tlrc` — symlink as needed if you want `tldr foo` to
work).

## Basic usage

```bash
# first run: pull the page cache (~3 MB)
tlrc --update

# look up a command (defaults to your platform with
# common fallthrough)
tlrc tar
tlrc git rebase           # multi-word page name
tlrc -p linux iptables    # force a specific OS section
tlrc -p osx launchctl     # macOS-specific page

# list every page available for a platform
tlrc --list -p macos | head

# render in a different language (community-translated)
tlrc -L zh tar

# bypass the cache and re-render a page from a local file
tlrc --render ./my-custom-page.md

# pipe-friendly: strip color and box drawing for grep
tlrc --no-color tar | grep -i compress
```

Configure themes, default platform, and cache TTL in
`~/.config/tlrc/config.toml` — `tlrc --config-path` prints
the location it's reading.

## When to choose

- **You want example-first command help, not a manual** —
  `man tar` documents every flag exhaustively; `tlrc tar`
  shows you the four invocations you actually wanted.
  This is the difference between "reference" and "recipe."
- **You're on a team that already standardized on
  `tldr-pages`** — using the upstream-official client
  (rather than a third-party clone) means cache layout,
  page format, and CLI flags match the documentation on
  `tldr.sh` exactly, and bug reports route to the same
  org that maintains the page corpus.
- **You want a stricter, more predictable cache** than
  `tealdeer` provides — SHA-256-verified downloads and
  lockfile-protected updates matter when `tlrc --update`
  runs from CI or a `topgrade` pass.
- **You want cross-language docs** — the `tldr-pages`
  corpus has community translations for ~30 languages,
  and `tlrc -L <code>` switches the rendered page
  language without a second cache.

## When NOT to choose

- **You're already happy with [`tealdeer`](../tealdeer/)**
  — `tealdeer` reads the same page format, has the same
  fundamental UX, and is faster at startup by a few
  milliseconds. The reasons to prefer `tlrc` are
  governance (official client) and cache-integrity
  features, not raw speed.
- **You want AI-generated explanations of unfamiliar
  commands** — `tldr-pages` is human-curated and bounded
  to commands the community has written pages for. For
  arbitrary natural-language → shell explanations, use
  [`tlm`](../tlm/), [`shell-gpt`](../shell-gpt/), or
  [`tgpt`](../tgpt/).
- **You need full reference manuals** — `man`, `info`,
  and `help2man` still own the "every flag, every
  option" niche. `tlrc` replaces "I forgot the syntax,"
  not "I need to understand every option."
- **You want offline pages without ever pulling a
  cache** — `tlrc` *can* run fully offline after the
  first `--update`, but the initial pull does require
  network. If you need a fully air-gapped setup, ship
  the `~/.cache/tlrc/` directory pre-populated.

## Why it fits the zoo

The zoo's recurring theme is "the canonical Unix
incantation, replaced with something denser and faster."
`tldr-pages` is the canonical replacement for "I just
want one example of this command," and `tlrc` is the
upstream-endorsed Rust client for that corpus. The zoo
already documents [`tealdeer`](../tealdeer/) as the
de-facto Rust client; adding `tlrc` covers the official
governance angle and gives readers the comparison they
need to pick between the two. Sits naturally next to
[`tldr`](../tldr/) (the original Node client),
[`navi`](../navi/) (interactive cheat-sheet TUI),
[`pet`](../pet/) (personal snippet manager), and
[`fend`](../fend/) (calculator-as-CLI) under the
"reduce-the-time-from-question-to-answer" cluster.

## Upstream pointers

- Repo: <https://github.com/tldr-pages/tlrc>
- Release notes: <https://github.com/tldr-pages/tlrc/releases>
- License: [MIT](https://github.com/tldr-pages/tlrc/blob/main/LICENSE)
- Page corpus: <https://github.com/tldr-pages/tldr>
- Web mirror: <https://tldr.inbrowser.app/>
- Comparison vs. `tealdeer`:
  <https://github.com/tldr-pages/tlrc#why-another-client>
