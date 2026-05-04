# bombadillo

> **Multi-protocol smol-net browser for the terminal** — a
> single Go binary that speaks **Gemini, Gopher, Finger, Local,
> and HTTP/HTTPS** (the last via a configurable text-mode
> handler like `lynx` / `w3m`) from one keyboard-driven TUI:
> link-numbered text view, vim-style modal editing (`:` command
> mode, `b` back, `B` forward, `g` go, `s` search), bookmark +
> certificate stores, source-view, save-to-disk, and a
> per-protocol theming layer — pinned to **v2.3.3** (commit on
> `master`,
> [LICENSE](https://tildegit.org/sloum/bombadillo/raw/branch/master/LICENSE),
> GPL-3.0-only).

Source: <https://tildegit.org/sloum/bombadillo>
(GitHub mirror: <https://github.com/sloumdrone/bombadillo>)

## TL;DR

The "smol web" — Gemini (`gemini://`), Gopher (`gopher://`),
Finger (`finger://`), and the small-text-page corner of HTTP
— is a small but persistent ecosystem of text-first content
(personal `gemlogs`, mailing-list archives, weather services,
classic Gopher holes still maintained at universities, the
Project Gutenberg Gopher mirror, the entire `gemini.circumlunar.space`
hub). The catalog already ships [`amfora`](../amfora/) for
Gemini-only browsing; `bombadillo` is the orthogonal pick:
**one client for every smol-net protocol at once**, with a
keyboard surface modelled directly on `vim` instead of `less`.

The killer property is **protocol parity in one process**:
`b gemini://gemini.circumlunar.space/`, `b gopher://gopher.floodgap.com/`,
and `b finger://happynetbox.com/` all open in the same buffer
with the same keymap, the same bookmark store, and the same
TLS fingerprint pin store (`~/.bombadillo.ini` +
`~/.config/bombadillo/`). Switching from "what does this
gemlog say?" to "is the Floodgap Gopher proxy still up?" to
"who is logged into this Unix host right now?" is one `:b`
command, not three different binaries.

## Install

```bash
# Source (any platform with a Go toolchain)
git clone https://tildegit.org/sloum/bombadillo
cd bombadillo
make
sudo make install                  # /usr/local/bin/bombadillo

# Arch Linux (AUR)
yay -S bombadillo

# Homebrew tap (community)
brew install --HEAD sloumdrone/bombadillo/bombadillo

# Pre-built binaries are also published for the v2.3.3 tag
# on the tildegit releases page.
```

Single static Go binary, no runtime dependencies. Works on
Linux, macOS, BSD; Windows builds via WSL.

## Example usage

```bash
# launch the TUI
bombadillo

# open a Gemini capsule directly
bombadillo gemini://gemini.circumlunar.space/

# open a Gopher hole
bombadillo gopher://gopher.floodgap.com/

# look up logged-in users on a Unix host
bombadillo finger://happynetbox.com/

# read a local Gemini / Gopher / Markdown file
bombadillo local:///home/user/notes/draft.gmi
```

In-TUI keymap (subset, vim-shaped):

- `j` / `k` scroll line, `Ctrl+F` / `Ctrl+B` page
- `g` / `G` top / bottom of buffer
- `1` … `9` follow numbered link; `[number]` for >9
- `b` / `B` back / forward in history
- `:b <url>` open URL in current buffer
- `:a <name>` add bookmark; `:bookmarks` browse them
- `:s <query>` search engine query (configurable)
- `:writeout <path>` save current page to disk
- `:source` view raw page source
- `:q` quit

## Why it matters

Gemini and Gopher are *not* dead niches — Gemini in
particular has had steady ~30 % year-over-year growth in
indexed capsules since 2020 (per `gemini://geminispace.info/`
crawl stats), and the Project Gutenberg Gopher mirror, the
SDF Public Access UNIX System, and a long tail of personal
`gemlogs` are read every day. The protocols are deliberately
simple (Gemini is ~50 lines of spec, Gopher predates HTTP),
content travels uncompressed text without ad networks /
trackers / JavaScript / cookies / fingerprinting / paywalls,
and a 256-KB page is large.

`bombadillo` is the right shape for that ecosystem: one
keyboard-only client that opens every protocol the
smol-net cares about, runs over SSH on an 80 × 24 tty,
and produces deterministic plain-text output an LLM agent
can ingest cleanly when the task is "summarise the latest
post on this gemlog" or "what's on the front page of
this Gopher hole today."

Pick over [`amfora`](../amfora/) when you also want Gopher /
Finger / source-view / vim-shaped keymap (amfora is
Gemini-only, mouse-friendly Bubble Tea UX, `less`-shaped);
pick over `lynx` / `w3m` when smol protocols matter (those
two are HTTP-first and either don't speak Gemini at all or
do so through external converters); pick over a browser
extension when air-gapped / no-X / no-browser is the
constraint (the in-flight laptop, the corp box, the
SSH-only dev box).

Pairs with [`amfora`](../amfora/) for per-protocol depth
(many operators run both — `amfora` for daily Gemini
reading, `bombadillo` when a Gopher-only resource appears),
with [`circumflex`](../circumflex/) (Hacker News reader of
the same shape — terminal-only, keyboard-driven), with
[`tuifeed`](../tuifeed/) / [`newsboat`](../newsboat/) (those
read RSS / Atom feeds; `bombadillo` reads the underlying
Gemini capsules many of those feeds describe), and
orthogonal to [`elinks`](https://github.com/rkd77/elinks)
or [`browsh`](https://www.brow.sh/) on the HTTP-heavy axis
(those render real HTML / CSS / JS in the terminal —
`bombadillo`'s HTTP support is text-mode only by design).

## License

GPL-3.0-only. See
[LICENSE](https://tildegit.org/sloum/bombadillo/raw/branch/master/LICENSE)
in upstream. The GPL-3.0 obligations apply to redistribution
of `bombadillo` itself; visited content carries its
authors' own licences.

## As of

2026-05-04. Upstream version `2.3.3` (the in-source `version`
constant on `master`; the tildegit Anubis JS challenge
sometimes blocks the releases listing — the binary's own
`bombadillo -v` is the authoritative version string at
runtime). The protocol surface (Gemini 1.0, Gopher RFC 1436,
Finger RFC 1288) has been stable for years; bombadillo's
keymap and `~/.bombadillo.ini` schema may evolve — re-check
the upstream `MANUAL.md` before pinning in scripted contexts.
