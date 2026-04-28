# atac

- **Repo:** https://github.com/Julien-cpsn/ATAC
- **Version:** v0.23.0 (latest stable, February 2026)
- **License:** MIT ([LICENSE](https://github.com/Julien-cpsn/ATAC/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install atac` · `cargo install atac` · `pacman -S atac` · static binaries on the GitHub release page · binary name is `atac` (Arguably a Terminal API Client)

## What it does

`atac` is a full Postman-style HTTP API client that runs entirely
inside the terminal. It gives you **collections**, **requests**,
**environments / variables**, **request history**, **pre- and
post-request scripts**, **cookie jars**, and an **authentication
panel** (Basic, Bearer, OAuth-style) — all in a keyboard-driven TUI.
Collections and environments are stored as plain files on disk
(JSON, or Postman v2.1 collection format) so you can `git`-track
them, share them with a team, and import existing Postman /
Insomnia / OpenAPI collections directly. Each request supports the
usual surface: method, URL with `{{var}}` interpolation, headers,
query params, body (raw, JSON, form-data, multipart, file), TLS
toggles, redirect policy, and response viewing with syntax
highlighting and pretty-printing.

## When to pick it / when not to

Pick `atac` when you've been using Postman or Insomnia and you want
the same workflow — saved requests, environments, history, scripts —
without leaving the terminal, without a per-seat login, and with
the collection living in your repo instead of in a cloud workspace.
It is also a good fit when you SSH into a jump host and need a
real API client there: it works on any system that can run a Rust
binary and a TTY. The Postman v2.1 import means you can hand a
designer's collection to your CI box and exercise it from there.

Skip it for **single-shot exploratory requests** — [`curlie`](../curlie/),
[`xh`](../xh/), or [`httpie`](../httpie/) are faster to type. Skip
it for **scripted multi-step flows with assertions** that need to
run in CI — [`hurl`](../hurl/) is purpose-built for that and
expresses the flow in a versionable text file. Skip it for **load
testing** ([`oha`](../oha/), `wrk`, `vegeta`) and for protocols
beyond HTTP/HTTPS (no native gRPC, GraphQL beyond raw POST, or
WebSocket UI).

## Why it matters in an AI-native workflow

API exploration is a common phase of agent-driven development:
discover an endpoint, capture working headers, save a request,
re-run it after a code change. Doing this with `curl` snippets in
chat history loses fidelity (auth tokens get stale, params get
mistyped, environments leak across requests). `atac` stores every
request as a structured file under version control, which means an
agent can edit a request the same way it edits source code, and the
next agent (or human) opening the repo gets exactly the same
working invocation. Because environments are explicit named files,
the agent can switch between `dev`, `staging`, and `prod` by
changing one line, with no risk of an in-memory variable leaking
the wrong base URL into a destructive call.

## Example invocations

```bash
# Launch the TUI with the default collection store (~/.config/atac)
atac

# Open a specific directory of collections (good for repo-tracked APIs)
atac --directory ./api-collections

# Import an existing Postman v2.1 collection on first launch:
# inside the TUI -> Collections panel -> 'i' -> point at the JSON file

# Typical flow inside the TUI:
#   c   create a collection
#   n   new request inside the selected collection
#   e   edit the request (method / URL / headers / body / auth)
#   Enter   send the request
#   v   toggle response view (pretty / raw / headers / cookies)
#   E   open the environments panel; set base_url, token, etc.
#   {{var}}  reference an env var anywhere in URL / headers / body
#   h   request history; r   re-send a past request
#   /   fuzzy-find a request across all collections

# Run from a CI step against a chosen environment file (headless mode)
# -- see `atac --help` for the current non-TUI invocation flags

# Files on disk are JSON; diff them in code review like any other source
git diff api-collections/payments/charge.json
```
