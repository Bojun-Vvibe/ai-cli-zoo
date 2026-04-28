# slumber

- **Repo:** https://github.com/LucasPickering/slumber
- **Version:** v5.2.5 (latest stable, 2026-04)
- **License:** MIT ([LICENSE](https://github.com/LucasPickering/slumber/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install slumber` · `cargo install slumber` · `cargo binstall slumber` · binary releases on the GitHub release page

## What it does

`slumber` is a terminal HTTP/REST client built around a YAML "collection"
file that lives in your repo and diffs cleanly in code review — Postman /
Insomnia ergonomics with a git-friendly source format and a TUI front end.
A `slumber.yml` declares `profiles:` (per-environment variable bags — `dev`,
`staging`, `prod`, with a `selected` default), `chains:` (named recipes that
extract a value from another request, a file, an env var, a shell command,
or a prompt — the standard "log in, capture the bearer token, reuse it on
every other call" loop becomes one declarative chain), and `requests:` (a
nested tree of recipes with `method`, `url`, `headers`, `body`, `query`, and
templated `{{var}}` / `{{chains.token}}` / `{{env.HOST}}` interpolation).
Launch the TUI (`slumber`) and you get a three-pane view: collection tree on
the left, recipe editor top-right, response (pretty-printed JSON / XML / HTML
with syntax highlighting, plus a raw view and a header view) bottom-right;
history is per-recipe and persists across sessions in a local SQLite store
(`~/.local/share/slumber/state.sqlite`) so yesterday's response is one
keypress away. The CLI mode (`slumber request <recipe-id>`) is the headless
twin: same collection, no TUI, output to stdout, exit code reflects HTTP
status — so the same `slumber.yml` drives interactive debugging *and* CI
smoke tests without a separate Newman / `curl`-script translation layer.
Authentication primitives are first-class (basic, bearer, AWS SigV4 via
chains, OAuth via the prompt chain), and the renderer surfaces certificate
chains and TLS errors in plain English instead of OpenSSL hex.

## When to pick it / when not to

Pick `slumber` when your team's API collection is currently a Postman export
nobody can review, or a directory of `curl` shell scripts with copy-pasted
bearer tokens. The YAML collection format is the differentiator — it lives
in the API repo, gets reviewed in PRs, supports `!include` for splitting big
collections, and templates against per-environment profiles instead of the
"export → import → re-edit URLs" Postman ritual. Chains solve the awkward
"call A, capture `id`, feed into B" workflow with one block of YAML rather
than a JavaScript pre-request script. Pair with [`xh`](../xh/) when you only
need a one-shot curl-shaped command and not a persistent collection — the
two coexist; `slumber` is the saved-state tier, `xh` is the ad-hoc tier.
Pair with [`jq`](../jq/) / [`gron`](../gron/) for response post-processing
in CLI mode (`slumber request get_user | jq '.email'`).

Skip it when the team is already happy with Postman / Insomnia / Bruno and
the friction is not the format — switching tools for tool-switching's sake
is rarely worth it. Skip it for raw load-testing (`k6`, `vegeta`, `oha` are
the right layer; `slumber` is functional, not load-shaped). Skip it for
gRPC / WebSocket / SSE-heavy APIs — `slumber` is HTTP/1.1 + HTTP/2
request/response shaped; long-lived bidirectional streams want
`grpcurl` / `wscat` / `websocat`. For browser-side API discovery (capturing
real traffic from a SPA), use the browser devtools or `mitmproxy` — `slumber`
is a writing tool, not a sniffer.

## Example invocations

```bash
# Open the TUI against ./slumber.yml in cwd
slumber

# Open against an explicit collection
slumber -f ./api/slumber.yml

# Headless: run one recipe, print the response body to stdout
slumber request get_user

# Override the active profile from the CLI (dev|staging|prod)
slumber request get_user --profile staging

# Override a single template variable for one call
slumber request create_widget --override widget_name=foo

# Print the response body to stdout, headers to stderr, JSON-pretty-printed
slumber request list_widgets --output body | jq '.[] | .id'

# Dry-run: show the assembled HTTP request without sending it
slumber request create_widget --dry-run
```

Minimal `slumber.yml` for reference:

```yaml
profiles:
  dev:
    default: true
    data:
      host: https://api.dev.example.test
      user: alice
  prod:
    data:
      host: https://api.example.test
      user: alice

chains:
  bearer_token:
    source: !request
      recipe: login
      trigger: !expire 1h
    selector: $.access_token

requests:
  - !folder
    name: Auth
    children:
      - !request
        id: login
        method: POST
        url: "{{host}}/auth/login"
        body: !json
          username: "{{user}}"
          password: "{{env.API_PASSWORD}}"

  - !folder
    name: Widgets
    children:
      - !request
        id: list_widgets
        method: GET
        url: "{{host}}/widgets"
        headers:
          Authorization: "Bearer {{chains.bearer_token}}"

      - !request
        id: create_widget
        method: POST
        url: "{{host}}/widgets"
        headers:
          Authorization: "Bearer {{chains.bearer_token}}"
        body: !json
          name: "{{widget_name}}"
```
