# httpie

> **A human-friendly HTTP client for the terminal** — `curl`'s
> Python-powered cousin that turns ad-hoc API debugging into a
> pleasant one-liner. Pinned to **v3.2.4** (commit
> `2105caa49bae87c5809c274e407619a0de2639d1`,
> [LICENSE](https://github.com/httpie/cli/blob/master/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/httpie/cli>

## TL;DR

`httpie` (binary name: `http`, plus `https` and `httpie`) is what
you reach for when you want to poke a JSON API from the shell and
do *not* want to remember `curl -H 'Content-Type: application/json'
-d '{"key":"value"}' -X POST`. The same call is `http POST
api.example.com/things key=value` — querystrings, JSON bodies, and
form fields all get inferred from `key=value`, `key==value`,
`key:=number`, and `key@file` syntax. Output is colour-highlighted
JSON by default, with sensible header / body separation and a
progress bar for large downloads. Plugins (auth, transport, format)
are pip-installable. There is a sister project `httpie-desktop` and
the now-separate `httpx` / `httpie cli` split — this entry tracks
the canonical `http` CLI.

## Install

```bash
# Homebrew (macOS / Linux)
brew install httpie

# Pip (any OS with Python ≥ 3.9)
python3 -m pip install --user 'httpie==3.2.4'
# or pipx for an isolated install
pipx install 'httpie==3.2.4'

# Linux distro packages
# Debian / Ubuntu: apt install httpie
# Fedora:         dnf install httpie
# Arch:           pacman -S httpie

# verify
http --version    # 3.2.4
```

## Representative examples

```bash
# 1. GET with a querystring + custom header — note `==` for query, `:` for header
http GET api.github.com/repos/httpie/cli \
     per_page==1 \
     'User-Agent:bojun-cli-zoo'

# 2. POST a JSON body without escaping a single quote
http POST httpbin.org/post \
     name=ada \
     age:=30 \
     tags:='["sre","backend"]' \
     bio@./bio.md

# 3. Download a large file with progress + resume
http --download --continue \
     https://releases.ubuntu.com/24.04/ubuntu-24.04.1-live-server-amd64.iso
```

## When to use vs. alternatives

- Pick **httpie** when the request is short, you want readable
  coloured output, and you are exploring an API by hand.
- Pick [`curlie`](../curlie/) (sister tool, same syntax) when you
  need `curl`'s exact wire-level behaviour but `httpie`'s ergonomics
  — `curlie` is a thin Go wrapper around `curl` itself, so every
  TLS/HTTP/2/3 quirk of `curl` is preserved.
- Pick `curl` when scripting in CI or copy-pasting from browser
  DevTools "Copy as cURL" — it is everywhere and stable.
- Pick [`xh`](../xh/) when you want `httpie` syntax in a single
  static Rust binary with no Python runtime — startup is ~10×
  faster, ideal for tight shell loops.
- Pick [`posting`](../posting/) when the workflow has graduated
  from one-liners into a saved collection (Postman-style) and you
  want a TUI.
