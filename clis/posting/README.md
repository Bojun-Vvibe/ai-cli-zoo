# posting

> **The modern API client that lives in your terminal** — a Textual TUI
> for HTTP / REST / GraphQL exploration that competes with Postman and
> Insomnia from inside `tmux` / SSH, with collections stored as
> human-readable YAML in git instead of locked behind a SaaS account.
> Pinned to **v2.10.0** (commit
> `56703a11513e8e74e681b4f859f31945b71e746f`,
> [LICENSE](https://github.com/darrenburns/posting/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/darrenburns/posting>

## TL;DR

`posting` is the keyboard-first API client the terminal has been
missing. `posting` opens a multi-pane Textual UI: collection tree on
the left, request editor (method / URL / params / headers / body /
auth / scripts) in the middle, response viewer (status + headers +
syntax-highlighted body + cookies) on the right; `Ctrl+J` sends the
request, `Ctrl+T` opens a new request tab, `Ctrl+P` is a fuzzy command
palette over every action, `Ctrl+S` saves the request to disk as
YAML. Requests live in a `--collection <dir>` of `.posting.yaml`
files that diff cleanly in git, so a team's API surface is a normal
PR instead of a Postman cloud workspace; environment variables come
from `.env` files plus a typed `*.env.yaml` overlay so
`{{API_KEY}}` interpolates correctly across staging / prod without
copying secrets into the request body.

## Install

```bash
# uv (recommended — fast isolated install of a Python TUI)
uv tool install posting

# pipx
pipx install posting

# pip
pip install posting

# verify
posting --version            # 2.10.0
posting --collection ./api   # open a collection directory
```

Single Python package, no native deps; runs anywhere a modern
Python + a true-color terminal does (macOS Terminal / iTerm2 /
Alacritty / WezTerm / Windows Terminal / `tmux`).

## One Concrete Example

A repo with an `api/` directory of YAML requests:

```bash
# 1. point posting at the collection
posting --collection ./api --env ./api/staging.env

# 2. inside the TUI:
#    Ctrl+P  → "new request"
#    type    → POST https://api.example.com/v1/chat/completions
#    Body    → JSON, paste {"model": "{{MODEL}}", "messages": [...]}
#    Headers → Authorization: Bearer {{OPENAI_API_KEY}}
#    Ctrl+J  → send; response pane shows status, headers, JSON body
#    Ctrl+S  → save as api/chat/completions.posting.yaml

# 3. the saved file is plain YAML, reviewable in a PR:
cat api/chat/completions.posting.yaml
# name: completions
# method: POST
# url: "{{BASE_URL}}/v1/chat/completions"
# headers:
#   - name: Authorization
#     value: "Bearer {{OPENAI_API_KEY}}"
# body:
#   content: '{"model": "{{MODEL}}", "messages": [...]}'

# 4. headless one-shot in CI (no TUI)
#    (posting is TUI-first; for CI use httpie / curl against the same YAML)
```

## Niche It Fills

**The keyboard-first API client for the terminal era.** Postman /
Insomnia / Bruno are GUI apps that assume an X11 / Wayland / macOS
desktop and store collections in a vendor-specific format (Postman
Cloud, Insomnia's sync product, Bruno's git-friendly but
proprietary `.bru`). `posting` is a Textual TUI that runs over SSH,
inside `tmux`, on a server with no display, and stores collections
as plain YAML you check into the same repo as the code that calls
the API. For the operator who is already inside a terminal session
debugging a service, opening a separate desktop app to fire one
test request is friction; `posting` removes it.

## Vs Already Cataloged

- **Vs raw `curl` / `httpie`:** `curl` / `httpie` are one-shot
  composers (one request per shell invocation); `posting` is a
  persistent multi-tab editor with history, environments, response
  pretty-printing, and a command palette. Use `curl` in a script;
  use `posting` for the interactive exploration phase before you
  bake the request into the script.
- **Vs Postman / Insomnia:** GUI apps that need a desktop and a
  vendor account for sync; `posting` runs in a terminal and uses
  git as the sync layer.
- **Vs [`elia`](../elia/) / [`oterm`](../oterm/):** those are TUI
  *chat* clients for LLMs; `posting` is a TUI *HTTP* client for
  any API. They cover different layers of the same workflow —
  use `posting` to debug your model provider's REST surface, then
  `elia` to chat against the same endpoint.

## Caveats

- TUI-first; **no headless one-shot mode** for CI runs (`posting`
  expects a TTY). For "run this `.posting.yaml` request in CI",
  pair with `httpie` / `curl` against the same env vars, or
  convert the YAML in a small script.
- The YAML format is `posting`-specific; Postman / Insomnia
  collections do not import directly. The maintainer ships a
  manual conversion guide; budget time for a one-time migration
  if you are leaving Postman.
- Scripting hooks (`pre_request` / `post_response` Python blocks)
  execute arbitrary Python from the YAML file — treat collections
  the same as any other code in your repo for review purposes.
- The TUI assumes a true-color terminal with at least 80 columns;
  the request editor pane gets cramped under ~100 cols, and
  legacy terminals without 256-color support render the syntax
  highlighting unreadably.
