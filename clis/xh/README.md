# xh

> **Friendly and fast HTTP client** — a single Rust binary that
> reimplements the `httpie` UX (intuitive `xh GET example.com
> name=value` syntax for query / header / JSON-body / form fields,
> coloured + pretty-printed JSON / XML / HTML responses, sessions,
> downloads with progress) as a static binary that starts in
> single-digit milliseconds, has no Python runtime dependency, and
> respects existing `~/.netrc`. Pinned to **v0.25.3** (commit
> `54387b5cfee5d7f97730d74230f311262527ba45`,
> [LICENSE](https://github.com/ducaale/xh/blob/master/LICENSE),
> MIT).

Source: <https://github.com/ducaale/xh>

## TL;DR

`xh` is what you reach for when `curl`'s `-H 'Content-Type:
application/json' -d '{"name":"alice"}'` ergonomics start eating
your day and `httpie`'s 1-second Python cold start (or its
"requires `pip` and a Python runtime on this server") becomes
friction. The syntax is the `httpie` syntax — `xh post
api.example.com/users name=alice email=alice@example.com
'X-API-Key:abc123'` does the right thing (HTTP method positional,
`key=value` is JSON body field, `key:=value` is non-string JSON,
`key==value` is query param, `Header:value` is request header,
`@file` reads a body / field from disk) — but it ships as a single
~6 MB Rust binary, so cold-starts in 5-15 ms, runs on a busybox
container with no Python, and feels native on Windows where
`httpie` is heavier. Output is coloured + pretty-printed by
default (JSON indented, headers highlighted, content type
auto-detected); `--download` saves to a file with a progress bar;
`--session=name` persists cookies / auth / hostname between calls.

## Install

```bash
# Homebrew (macOS / Linux)
brew install xh

# Cargo
cargo install --locked xh

# Linux package managers
# Arch: pacman -S xh
# Nix: nix-env -iA nixpkgs.xh
# Debian / Ubuntu: download .deb from releases page

# Windows
# scoop install xh
# choco install xh
# winget install xh

# from a release tarball (any OS)
curl -Lo xh.tar.gz "https://github.com/ducaale/xh/releases/download/v0.25.3/xh-v0.25.3-aarch64-apple-darwin.tar.gz"
tar xf xh.tar.gz --strip-components=1
sudo install xh /usr/local/bin/

# verify
xh --version    # xh 0.25.3
```

`xh` will detect a missing `https://` and add it; it will detect
JSON-shaped fields and set `Content-Type: application/json`
automatically; it will pretty-print the response if stdout is a
TTY and stream raw if it is a pipe. Zero config required.

## License

MIT — see [LICENSE](https://github.com/ducaale/xh/blob/master/LICENSE).
Permissive, no attribution required for binaries; redistribute
freely.

## One Concrete Example

```bash
# 1. quick GET with query params + a custom header
xh httpbin.org/get foo==bar 'X-Trace-Id:abc-123'
# HTTP/1.1 200 OK
# content-type: application/json
# {
#   "args": { "foo": "bar" },
#   "headers": { "X-Trace-Id": "abc-123", ... },
#   ...
# }

# 2. POST a JSON body (key=value is string, key:=value is non-string)
xh post httpbin.org/post \
  name=alice \
  age:=30 \
  tags:='["admin","staff"]' \
  active:=true \
  'Authorization:Bearer $TOKEN'

# 3. multipart form upload
xh --form post httpbin.org/post \
  username=alice \
  avatar@~/Pictures/me.png

# 4. session: log in once, reuse cookies / auth across calls
xh --session=demo post api.example.com/login \
  username=alice password=secret
xh --session=demo get api.example.com/me        # cookies + auth replayed

# 5. download a file with a progress bar
xh --download https://example.com/dataset.tar.gz

# 6. pipe a JSON file in as the body
cat payload.json | xh post api.example.com/ingest

# 7. raw output for scripting (no colour, no pager)
xh --pretty=none get api.example.com/v1/users | jq '.[].id'

# 8. follow redirects (off by default for safety) and only print the body
xh --follow --body get bit.ly/some-shortlink
```

## Niche It Fills

**`httpie`-grade ergonomics with `curl`-grade portability.** For
ad-hoc HTTP debugging — exploring a new API, replaying a request
with a tweaked header, scripting a quick health-check — `httpie`'s
syntax is decisively better than `curl`'s, but `httpie` is a
Python install with cold-start cost and a runtime dependency that
is awkward on minimal containers, BusyBox SSH targets, and CI
images that do not already ship Python. `xh` keeps the syntax and
trades the runtime for a single static binary; the tax was 6 MB on
disk and zero on cold-start latency.

## Why use it

Three things `xh` does that `curl` does not, that pay back the
switching cost:

1. **JSON-by-default with shorthand.** `name=alice email=...` is a
   JSON body, `key:=value` is a non-string JSON value, `key==value`
   is a query param — no quoting, no `-H 'Content-Type:
   application/json'`, no manual `jq` to assemble the body. For
   the 80% of API debugging that is "POST some JSON, look at the
   JSON response," this collapses three flags into one positional.
2. **Pretty-printed coloured output by default.** Response JSON is
   indented and key-coloured; XML / HTML are syntax-highlighted;
   the status line + headers are bold; binary content is detected
   and not dumped to your terminal. `curl | jq` gives you the
   first half; `xh` makes it the default.
3. **Sessions persist auth / cookies / hostname.** `--session=foo`
   stores cookies, `Authorization`, and the resolved host in
   `~/.config/xh/sessions/foo.json`; subsequent `xh --session=foo
   <path>` calls replay them. For exploring an authenticated API
   you log in once and the next 50 calls just work — `curl
   --cookie-jar` does this but you have to hand-thread it.

For an LLM-CLI workflow where you are scripting "agent calls API,
parses response, decides next action," `xh --pretty=none --body`
is a clean stdin / stdout primitive that pipes to `jq` /
[`mods`](../mods/) / [`llm`](../llm/) without escaping nightmares.

## Vs Already Cataloged

- **Vs [`posting`](../posting/):** orthogonal — `posting` is a
  multi-pane Textual TUI for *interactive* HTTP API exploration
  with collections-as-YAML, environments, and per-request scripts;
  `xh` is a one-shot CLI for `curl`-shaped scripting and ad-hoc
  debugging. Pick `posting` for "I'm exploring a new API and want
  history + tabs + saved requests"; pick `xh` for "I want one
  request in a shell pipe."
- **Vs [`oterm`](../oterm/) / [`elia`](../elia/):** orthogonal —
  those are TUI chat clients for talking to LLMs, `xh` is a TUI-
  free HTTP client for talking to any HTTP API. Different stacks.
- **Vs `curl` (not cataloged):** `curl` is everywhere, has decades
  of muscle memory, supports more protocols (SFTP, IMAP, RTSP),
  and is the right answer for "this needs to run on a 1995 BusyBox
  router." `xh` is the right answer everywhere else: nicer JSON
  ergonomics, sessions, prettier defaults, and a smaller cognitive
  load. Keep `curl` for protocol breadth and obscure flags; reach
  for `xh` for everyday HTTP API debugging.
- **Vs `httpie` (not cataloged):** the closest peer — `xh` is
  *deliberately* compatible with `httpie`'s argument syntax, so
  muscle memory transfers. The differences are runtime (`httpie`
  needs Python, `xh` does not), startup latency (`httpie` ~700 ms
  cold, `xh` ~10 ms), and binary size (single static `xh` binary
  vs a `pip install` tree). Pick `httpie` if you want its plugin
  ecosystem (`httpie-jwt`, `httpie-aws-auth`); pick `xh` if you
  want the same syntax without the Python tax.

## Caveats

- **`httpie` plugin ecosystem is not portable.** `xh` does not
  load `httpie` plugins (e.g. AWS SigV4, JWT auth, OAuth2 helpers
  shipped as `httpie-*` packages). For complex auth flows you
  either pre-mint the token (`aws sts get-session-token`, paste
  the `Authorization` header) or stay on `httpie`. `xh` does
  support basic auth, bearer auth, digest auth, and `~/.netrc`
  natively.
- **HTTP/3 / QUIC support is partial.** `xh` is built on `reqwest`
  / `hyper` and inherits whatever HTTP/2 / HTTP/3 support those
  layers have at build time. For HTTP/3 testing prefer `curl`
  with a recent `curl --http3` build.
- **Output detection is heuristic.** `xh` looks at `Content-Type`
  to decide pretty-print mode; an API that returns JSON with
  `Content-Type: text/plain` will land as plain text. Force a
  format with `--pretty=all` / `--style=fruity` or pipe through
  `jq` directly.
- **Sessions store cookies + headers in plaintext** at
  `~/.config/xh/sessions/<name>.json` — fine for dev tokens, not
  for production credentials. Use `--auth-type=bearer --auth=...`
  per-call (no session) or pull the token from a secrets manager
  in a wrapper script for high-stakes use.
- **Default does not follow redirects** (intentional, mirrors
  `httpie`). Pass `--follow` for redirects or `--max-redirects N`
  for a bounded chase. The first time this surprises you, it is
  protecting you from accidentally hitting a different host.
- **Binary responses are detected and elided.** `xh` will print
  `+-----------------------------------------+` placeholders for
  binary bodies to keep your terminal sane; pass `--download` to
  save them or `--pretty=none --body` to dump the raw bytes
  through a pipe.
