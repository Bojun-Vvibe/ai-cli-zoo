# amfora

> **Terminal client for the Gemini protocol** — a fully
> keyboard-driven, tabbed browser for `gemini://` content
> with bookmarks, subscriptions (Atom/Gemini feeds),
> client-certificate identity management, and a Bombadillo-
> style multi-tab UI rendered with `tcell`. Pinned to
> **v1.11.0**
> ([LICENSE](https://github.com/makew0rld/amfora/blob/master/LICENSE),
> GPL-3.0-only).

Source: <https://github.com/makew0rld/amfora>

## TL;DR

`amfora` is a TUI browser for **Gemini**, a hand-built
Internet protocol that is deliberately smaller than HTTP:
one request, one response, mandatory TLS, no cookies, no
JavaScript, no redirects without consent, a text-mostly
markup called gemtext (`text/gemini`) where every link is
on its own line. The protocol is small enough that a full
client fits in one terminal binary; `amfora` is the most
feature-complete of those clients. Tabs (`Ctrl+T`,
`Ctrl+W`, number keys to switch) make multi-page reading
work the way it does in a graphical browser; bookmarks
(`Ctrl+B` to add, `Ctrl+D` to open the manager) and a
typed history (`Tab` completes from prior visits in the
URL bar) round out the navigation surface. Subscriptions
poll Gemini feeds and Atom feeds served over `gemini://`
on a configurable interval and surface new posts on a
synthetic `about:subscriptions` page — the closest the
protocol has to RSS reading. Per-host TOFU (trust on
first use) certificate pinning is on by default, with a
known-hosts file under `$XDG_DATA_HOME/amfora` that you
can audit and prune. Client-certificate identity (one
keypair per host, or one shared "persona" across hosts) is
how Gemini handles the equivalent of "logged-in" pages —
`amfora` generates and stores the keys for you and offers
them when a server requests one. Theme is fully
configurable via TOML (`config.toml`), proxies for non-
Gemini schemes (`gopher://`, `http(s)://`, `finger://`)
hand off to a configured external command, and the whole
thing builds to a single static Go binary with no runtime
dependencies.

## Install

```bash
# Homebrew
brew install amfora

# Arch Linux
sudo pacman -S amfora

# Single-binary download (GitHub releases)
curl -L -o amfora \
  https://github.com/makew0rld/amfora/releases/download/v1.11.0/amfora_1.11.0_linux_amd64
chmod +x amfora && sudo mv amfora /usr/local/bin/

# macOS arm64
curl -L -o amfora \
  https://github.com/makew0rld/amfora/releases/download/v1.11.0/amfora_1.11.0_darwin_arm64
chmod +x amfora && sudo mv amfora /usr/local/bin/

# Build from source (Go 1.17+)
git clone --depth 1 --branch v1.11.0 \
  https://github.com/makew0rld/amfora.git
cd amfora && go build .
```

## Usage

```bash
# Open a capsule (Gemini for "site")
amfora gemini://geminiprotocol.net/

# Start with no URL — drops into the bookmarks page
amfora

# Inside the TUI:
#   Ctrl+T new tab          Ctrl+W close tab     1..9 switch tab
#   Space   open URL bar    Tab    autocomplete  Enter follow link
#   Ctrl+B add bookmark     Ctrl+D bookmark mgr  Ctrl+A subscriptions
#   q       quit            Esc    cancel input  ?    help
```

Config lives at `$XDG_CONFIG_HOME/amfora/config.toml` (or
`~/.config/amfora/config.toml`). Common knobs:

```toml
[a-general]
home = "gemini://geminiprotocol.net/"
auto_redirect = false        # ask before following 3x
http = "default"             # or "off", or a shell command
search = "gemini://geminispace.info/search"
color = true

[subscriptions]
popup = true
update_interval = 1800       # seconds
```

## Why it's interesting

The Gemini protocol slot is small — it has to be, the
protocol fits in a 4-page spec — but two clients dominate
it: terminal (`amfora`, [`bombadillo`](https://tildegit.org/sloum/bombadillo),
`asuka`, `castor`) and graphical (`Lagrange`). Inside the
terminal slice, `amfora` is the one with **subscriptions
+ tabs + client certs + a written stable config format**;
`bombadillo` is older and also handles Gopher/Finger but
its UI is single-pane and command-driven (`b` for back,
`g` for go); `asuka` is a Rust minimalist with no
bookmarks or feeds; `castor` is GTK-only and not really a
peer. Pick `amfora` when (a) you're reading a corpus of
~50–500 Gemini capsules and want the same tab + bookmark
muscle memory you have in a graphical browser without
launching one, (b) you're publishing a Gemini capsule and
want a serious-feeling client to test against (TOFU,
client certs, slow-link rendering, `text/gemini` parser
edge cases), or (c) the appeal of Gemini for you is
specifically *the small protocol read in a small client*
and a Go binary that does one thing and exits cleanly fits
that aesthetic. Not the right pick if you actually want
the modern web — Gemini's whole premise is that it
deliberately isn't, no images inline, no styling, no
forms beyond a single-line input prompt; `amfora` honors
that and won't render an HTML page even when proxied.
Maintained since 2020, reached its 1.0 in 2021, on a slow
and deliberate release cadence (v1.11.0 is the current
tag), with the last few releases focused on TLS hygiene
(stricter cert validation, SNI fixes) and gemtext spec
conformance rather than new features — appropriate for a
client of a frozen protocol.
