# bruno

> **Git-friendly, fully-offline API client — collections live as
> plain `.bru` text files in your repo, no cloud sync, no
> account, with a CLI runner that drops cleanly into CI.** Pinned
> to **v3.3.0**,
> [license.md](https://github.com/usebruno/bruno/blob/main/license.md),
> MIT.

Source: <https://github.com/usebruno/bruno>

## TL;DR

`bruno` is the answer to "I want Postman / Insomnia, but the
collection should live in my git repo as readable text and the
runner should work in CI without a hosted account." Each request
is a `.bru` file (a small DSL — method, URL, headers, body,
assertions, scripts) committed alongside code; the desktop app
edits them visually, and `bru run` (the `@usebruno/cli` package)
executes one or many in CI with JUnit / JSON / HTML reporters.
There is no workspace cloud, no team sync server, no telemetry
backplane to opt out of — a Bruno collection is just a directory
of text.

## Install

```bash
# Desktop app (macOS / Linux / Windows) — visual collection editor
brew install --cask bruno
# or grab a .dmg / .deb / .AppImage / .exe from
# https://github.com/usebruno/bruno/releases/tag/v3.3.0

# CLI runner — what you actually want in CI
npm install -g @usebruno/cli      # global
npm install --save-dev @usebruno/cli  # per-repo, pinned in package.json

# verify
bru --version                     # 3.x.y from @usebruno/cli
bru run --help
```

## License

MIT — see
[license.md](https://github.com/usebruno/bruno/blob/main/license.md).
Permissive: ship the binary and the `.bru` files freely; the
collection format itself is plain text under your repo's own
license.

## One Concrete Example

A repo layout that ships an API contract + a CI smoke test:

```
my-service/
├── src/
├── api-tests/
│   ├── bruno.json              # collection metadata
│   ├── environments/
│   │   ├── local.bru           # baseUrl = http://localhost:8080
│   │   └── staging.bru         # baseUrl = https://staging.example.com
│   └── health/
│       ├── liveness.bru
│       └── readiness.bru
└── package.json
```

A request file (`api-tests/health/readiness.bru`):

```
meta {
  name: readiness
  type: http
  seq: 1
}

get {
  url: {{baseUrl}}/healthz/ready
}

headers {
  accept: application/json
}

assert {
  res.status: eq 200
  res.body.status: eq "ready"
  res.responseTime: lt 500
}

tests {
  test("checks pass count is non-zero", function() {
    expect(res.body.checks_passed).to.be.greaterThan(0);
  });
}
```

CI invocation:

```bash
# Run the collection against staging, fail the build on any
# non-2xx or any failed assert / test.
bru run api-tests \
  --env staging \
  --reporter-junit reports/bruno-junit.xml \
  --reporter-html  reports/bruno.html

# Run a single folder
bru run api-tests/health --env local

# Stream NDJSON for further processing
bru run api-tests --env staging --reporter-json - | jq '.summary'
```

## Niche It Fills

**Git-native, offline-first API client.** Postman and Insomnia
both default to cloud-synced workspaces and an account; even
their "local-only" modes still keep the collection in an opaque
SQLite / JSON blob inside an app data dir, which makes
code-review of an API change ("we added an `Idempotency-Key`
header to `POST /payments`") essentially impossible. Bruno's
`.bru` files are diffable text — a PR that touches a request
shows up in `git diff` exactly like a code change, reviewers
read it inline, and `bru run` in CI gives the same workflow a
real assertion runner. The desktop app is a thin editor over
the same files; you can bypass it entirely and edit `.bru` in
your IDE.

## Why use it

Three things `bruno` does that `postman` / `insomnia` /
`hurl` / `httpie` do not, all at once:

1. **Collection is plain text in the repo, no cloud.** No
   workspace ID to share, no "sign in to sync," no
   account-bound team. A new contributor who clones the repo
   already has the API contract — the `api-tests/` directory
   *is* the documentation. Postman's [Bruno Migrator] exists
   precisely because teams want to escape the cloud lock-in;
   Insomnia's git sync requires a paid plan and stores opaque
   YAML.
2. **Same artifact runs in the desktop app and in CI.** The
   `.bru` files Bruno's GUI writes are exactly what `bru run`
   executes. There is no "export collection, then run with
   newman" two-step (the way Postman → newman works) — write
   it once, run it everywhere. Environment files are also
   `.bru` text, so per-stage config is also reviewable.
3. **No telemetry, no account, no required network egress.**
   The desktop app does not phone home, the CLI does not phone
   home, and there is no hosted backplane to be on the wrong
   side of when it has an outage. The "open" in "Bruno is open
   source" includes the data plane, not just the source code.

For an LLM-CLI workflow that already lives in `git` and CI,
Bruno is the API client that does not fight the workflow.

## Vs Already Cataloged

- **Vs [`hurl`](../hurl/):** complementary; different shape.
  Hurl is a single-file declarative HTTP test format with a
  static binary runner — minimal, no GUI, perfect for "here is
  one `.hurl` file that exercises one endpoint." Bruno is a
  multi-request *collection* with environments, scripts,
  reusable folders, and a desktop editor for non-CLI
  teammates. Pick `hurl` for one-file checks driven from a
  Makefile; pick `bruno` when the collection is the API
  contract and the team includes people who want a GUI.
- **Vs [`curlie`](../curlie/) / [`xh`](../xh/):** unrelated.
  Those are interactive `curl` replacements for one-off
  requests. Bruno is the "save the request, version it,
  re-run it in CI" layer above them.
- **Vs [`atac`](../atac/):** both are open-source Postman
  alternatives, but `atac` is a TUI that operates on its own
  JSON-collection format inside `~/.config/atac` and targets
  "I want Postman in a terminal." Bruno targets "I want the
  collection in my repo as text plus a real CI runner." Pick
  `atac` for terminal-only ad-hoc work; pick `bruno` when the
  collection should be a repo artifact reviewed in PRs.
- **Vs [`grpcurl`](../grpcurl/):** different protocol surface.
  `grpcurl` is for gRPC; Bruno is HTTP / GraphQL / WebSocket /
  SSE / gRPC (gRPC support landed in 2.x). Pick `grpcurl` when
  the only protocol is gRPC and you want the lightest
  possible binary.

## Caveats

- **The desktop app is Electron.** It is a ~250 MB install
  with the usual Electron memory footprint. The CLI does not
  carry that weight (it's a normal Node package, ~40 MB
  installed) so CI is fine; the cost is on the workstation.
- **`.bru` is a custom DSL, not OpenAPI.** Bruno does not
  read or write OpenAPI specs natively; an importer exists
  (`bru import openapi …`) but the source-of-truth artifact
  is the `.bru` file, not the spec. If you want OpenAPI to
  *be* your collection format, this is the wrong tool —
  reach for an OpenAPI-driven runner instead.
- **Scripts run in a sandboxed JS VM, not full Node.** `vm2`
  / `isolated-vm` style isolation means most npm packages
  are not available inside `pre-request` / `post-response`
  scripts. For request signing / HMAC / OAuth dances that
  need real crypto, prefer environment variables populated
  by an outer shell step over inline scripts.
- **Bruno v3.x changed the `.bru` parser internals**;
  collections authored in 1.x mostly round-trip but a few
  edge cases (deeply-nested `body:json` with comments) need
  re-saving from the v3 GUI. Pin `@usebruno/cli` to a
  specific minor in `package.json` and bump intentionally.
- **No hosted run history / dashboard.** This is by design,
  but it means "show me the last 30 days of CI smoke
  results" is your CI system's job, not Bruno's. Pipe
  `--reporter-junit` into your existing test reporter.
