# wuzz

> **An interactive `curl`** — a Go TUI for crafting, sending, and
> dissecting HTTP requests one keystroke at a time, with editable
> URL / method / headers / body panes and a syntax-highlighted
> response view next to them.
> Pinned to **v0.5.0**
> ([LICENSE](https://github.com/asciimoo/wuzz/blob/master/LICENSE),
> AGPL-3.0).

Source: <https://github.com/asciimoo/wuzz>

## TL;DR

`wuzz` is what `curl` looks like if it had a UI. You launch it,
get a multi-pane terminal layout (URL, method, params, headers,
body, response headers, response body), tab between panes,
edit any of them inline, hit `Ctrl-R` to send the request, and
the response appears in the pane on the right with content-type-
aware syntax highlighting and a search overlay (`/`). Settings
can be persisted to `~/.wuzz/config.toml`; per-request "history"
is stored as JSON files you can replay or share. It is single
binary, no daemon, no config required to start.

## Install

```bash
# Go (any platform with Go 1.18+)
go install github.com/asciimoo/wuzz@latest

# Homebrew
brew install wuzz

# Linux package managers
# Arch:           pacman -S wuzz
# Alpine:         apk add wuzz
# Nix:            nix-env -iA nixpkgs.wuzz

# from source
git clone https://github.com/asciimoo/wuzz && cd wuzz
go build -o wuzz . && sudo install -m755 wuzz /usr/local/bin/

# verify
wuzz --version    # 0.5.0

# launch — drops you straight into the TUI
wuzz https://api.github.com/repos/asciimoo/wuzz
```

## License

AGPL-3.0 — see
[LICENSE](https://github.com/asciimoo/wuzz/blob/master/LICENSE).
Strong copyleft with the network-use clause: if you run a
modified `wuzz` as a network service, you must offer source to
the users of that service. For local CLI use this is irrelevant;
for embedding inside a SaaS surface, prefer one of the MIT-
licensed alternatives below.

## One Concrete Example

```bash
# 1. start with a URL pre-filled
wuzz https://httpbin.org/get

# inside wuzz — useful keys (full list in the docs / Ctrl-? overlay):
#   Tab / Shift-Tab     move between panes
#   Ctrl-R              send request
#   Ctrl-S              save response body to file
#   Ctrl-F              save current request as a history JSON
#   Ctrl-W              load a saved request
#   /                   search inside the response pane
#   Ctrl-Space          open method picker (GET/POST/PUT/PATCH/DELETE/…)
#   Ctrl-J / Ctrl-K     scroll response
#   Ctrl-C              quit

# 2. POST a JSON body — type into the body pane after switching to POST
#    (method picker = Ctrl-Space, then edit the body pane directly)

# 3. add custom headers — switch to the headers pane and type one per line:
#       Authorization: Bearer eyJhbGciOi…
#       Content-Type: application/json

# 4. replay a saved request from the shell
wuzz --request-file ~/.wuzz/history/2026-05-02-13-04-create-user.json

# 5. proxy through mitmproxy / Charles to inspect the wire bytes
HTTPS_PROXY=http://127.0.0.1:8080 wuzz https://api.example.test/v1/me

# 6. timeout and TLS knobs from the CLI
wuzz --timeout 5 --tls-skip-verify https://self-signed.example.test/
```

## Niche It Fills

**The "REPL for HTTP" gap between `curl` and a full GUI client.**
`curl` is perfect for one-shot scripted requests but punishing for
exploration: you cannot edit the headers of the request you just
sent without retyping the whole line, and reading the response
means piping into `jq`/`less`/`xxd` in another window. Full GUI
clients (Postman, Insomnia, Bruno) are great but heavy, modal,
and outside the terminal. `wuzz` sits in the middle: keyboard-
first, single binary, runs over SSH, but with editable panes
and a real response viewer instead of a teletype.

## Why use it

Three concrete things `wuzz` makes pleasant that the `curl`
loop does not:

1. **Iterative request shaping.** "Send the request, look at the
   response, change one header, send it again" is one keystroke
   per iteration in `wuzz` (edit pane, `Ctrl-R`). With `curl`
   it is up-arrow / Home / arrow-arrow-arrow / edit / Enter
   every time.
2. **Response inspection without leaving the tool.** JSON, HTML,
   and XML responses are syntax-highlighted in place and `/` is
   a working in-pane search. For a 4 KB JSON response that's a
   real productivity gain over `| jq | less` and a real safety
   gain over `| jq` (which fails opaquely on non-JSON).
3. **Session capture as a side-effect.** Every request you send
   is appended to history; `Ctrl-F` snapshots the current
   request as a JSON file you can commit to a repo as a
   reproducible bug repro or share with a teammate. No "hover
   over Postman, click ⋯, click Export, choose v2.1, …".

For an LLM-CLI workflow, `wuzz` is the **human review surface**
when an agent has drafted an API call: the human can paste the
URL/headers/body the agent proposed, send it once interactively
to confirm the response shape, then approve the agent's plan to
make the same call programmatically. `wuzz`'s saved-request JSON
is also a clean handoff format — agent generates it, human
inspects in `wuzz`, agent later replays via `--request-file`.

## Vs Already Cataloged

- **Vs [`oha`](../oha/) / [`plow`](../plow/) / [`ali`](../ali/):**
  Those are HTTP load generators — they hammer one URL at a
  configured rps and report latency distributions. Orthogonal
  to `wuzz`, which is a single-request exploration tool. You
  use `wuzz` to *figure out* what request to make, then `oha`
  to load-test it.
- **Vs [`xh`](../xh/) and [`httpie`](../httpie/):** Those are
  shell-pleasant single-line clients (`xh POST api.example.test/u
  name=alice`). Better than `wuzz` for "I already know exactly
  what request I want, type it once, get the response". Worse
  than `wuzz` for "I need to try eight variants of this header
  and watch the response change".
- **Vs [`posting`](../posting/):** Posting is a modern Textual-
  based TUI in the same family as `wuzz` with a richer
  request/collection model and a more polished UI. `wuzz` is
  the older, lighter, single-Go-binary alternative; pick
  `posting` for a Postman-style multi-collection workflow,
  pick `wuzz` for "drop one binary on a server I just
  SSH'd into and poke an API".
- **Vs [`hurl`](../hurl/):** Hurl is a *file-based* HTTP DSL
  for tests and CI ("send these requests, assert these
  responses"). Use Hurl when the request flow is locked in and
  you want it in version control; use `wuzz` when you are
  still figuring the flow out.
- **Vs [`mitmproxy`](../mitmproxy/):** mitmproxy *intercepts*
  traffic from another client; `wuzz` *originates* traffic
  itself. Pair them: send from `wuzz`, watch the wire in
  mitmproxy.

## Caveats

- **AGPL-3.0 license.** Fine for personal use, in-house tools,
  and SSH'ing into your own boxes. If you embed `wuzz` (or a
  fork) in a hosted service that exposes its UI to other users,
  the AGPL network clause kicks in and you must publish your
  source. For SaaS embedding, prefer `xh`/`httpie` (BSD/MIT) or
  `posting` (MIT).
- **Maintenance is intermittent.** v0.5.0 (2021) is the latest
  tagged release; the project is still on the original
  `jroimartin/gocui` widget toolkit, which itself is now
  archived. It works, but expect rare cosmetic glitches and
  no new features. For a more actively maintained
  request-shaping TUI, consider `posting`.
- **No first-class request collections.** History is a flat
  directory of JSON files; there is no "folder of related
  requests with shared environment variables" abstraction.
  If your workflow is "thirty endpoints in a Postman
  collection", you'll outgrow `wuzz` quickly.
- **No native cookie jar across sessions.** Cookies set in one
  request are remembered within the running `wuzz` process,
  but exiting and relaunching loses them. For session-heavy
  flows (login → many requests), drive cookies via headers
  explicitly.
- **TLS/cert UI is minimal.** `--tls-skip-verify` exists; pinning,
  client certs, mTLS, and inspecting the server cert chain do
  not. For TLS work, drop down to `curl -v` or `openssl s_client`.
