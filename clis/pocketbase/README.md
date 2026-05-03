# pocketbase

- **Repo:** https://github.com/pocketbase/pocketbase
- **Version:** v0.37.5 (2026-05-02)
- **License:** MIT ([LICENSE.md](https://github.com/pocketbase/pocketbase/blob/master/LICENSE.md))
- **Language:** Go
- **Install:** prebuilt binary from [GitHub releases](https://github.com/pocketbase/pocketbase/releases) · `go install github.com/pocketbase/pocketbase/examples/base@latest` · embed as a Go library

## What it does

`pocketbase` is a single-file open-source backend that bundles an embedded SQLite database, a REST + realtime (SSE) API auto-generated from your collection schemas, an auth service (email/password, OAuth2 providers, OTP), file storage (local filesystem or S3-compatible object store), and a polished admin UI — all into one ~30 MB Go binary that you `./pocketbase serve` and walk away from. The data model is collection-based (think Firestore "collections" or Airtable "tables"): you define collections through the admin UI or via migrations, each with a typed schema (text / number / bool / email / url / date / select / file / relation / json / editor), and the binary instantly exposes `GET/POST/PATCH/DELETE /api/collections/{name}/records` with filter/sort/expand query syntax (`?filter=status='active' && created>'2026-01-01'&sort=-created&expand=author`), plus `/api/realtime` SSE subscriptions that push record changes as they happen. Auth records live in dedicated "auth" collections; the API ships `/api/collections/users/auth-with-password`, `/api/collections/users/auth-with-oauth2`, password reset flows, email verification, and per-collection access rules expressed as JS-like expression strings (`@request.auth.id != "" && owner = @request.auth.id`) that PocketBase compiles to SQL `WHERE` clauses for row-level authorization. Customization is two flavors: (1) **JS hooks** — drop `pb_hooks/*.pb.js` files that the embedded `goja` JS runtime executes on lifecycle events (`onRecordCreateRequest`, `onMailerSend`, custom routes via `routerAdd("GET", "/hello", ...)`); (2) **Go extension** — import `github.com/pocketbase/pocketbase` as a library, register your own routes / event hooks / migrations, and `go build` your own single-binary backend that still ships the admin UI and stock APIs. Migrations live as plain `.go` or `.js` files under `pb_migrations/`, applied automatically on startup. Everything — schema, records, files, hooks, migrations — sits under one `pb_data/` directory you can `tar | scp` to back up or move between hosts.

## When to pick it / when not to

Pick `pocketbase` when you want the "Firebase developer experience" (auth + realtime DB + file storage + admin UI in one SDK call) without the SaaS lock-in, the egress fees, or the operational complexity of running Supabase / Appwrite / Parse Server stacks. Concrete cases: you are prototyping a side-project / MVP / internal tool and want users + records + uploads + a working admin panel by the end of the day, deployed as `./pocketbase serve` on a $5 VPS or a Fly.io machine; you are building a small SaaS where the entire backend genuinely fits on one box (typical PocketBase deployments comfortably handle thousands of concurrent SSE subscribers and hundreds of writes/sec on modest hardware before you need to think about sharding); you want a reproducible local dev backend that any teammate can `git clone && ./pocketbase serve` against without provisioning Postgres + Redis + S3 + an auth service; you are writing a Go program and want to embed PocketBase as a library to ship a fat-binary product (desktop app with backend, on-prem appliance, edge device) where the backend "is" the product. Pair with [`litestream`](../litestream/) (continuous SQLite replication to S3 — solves the "single SQLite file is my whole DB" durability concern), [`caddy`](../caddy/) (TLS reverse proxy in front of `pocketbase serve`), [`rclone`](../rclone/) (back up `pb_data/` to any cloud), and [`mkcert`](../mkcert/) (local TLS during dev). For schema-as-code workflows pair with [`sqlite-utils`](../sqlite-utils/) for ad-hoc inspection and [`atlas`](../atlas/) is overkill here — PocketBase has its own migration system.

Skip `pocketbase` when your write throughput or dataset already needs Postgres / MySQL semantics (multi-writer, true concurrent writes beyond SQLite's WAL serialization, partitioning, logical replication, FDWs, materialized views) — go to [`supabase-cli`](../supabase-cli/) / [`hasura-cli`](../hasura-cli/) / Postgres directly. Skip when you need horizontal scale-out (multi-node, multi-region, active-active) — PocketBase is intentionally a single-process design, and the right answer is a different stack, not "ten PocketBase nodes". Skip when your team's strict requirement is RBAC + audit-grade authz + SSO / SAML / SCIM + per-tenant data isolation at scale — PocketBase's per-collection rule strings are good for app-level row-security but are not a substitute for an enterprise IAM stack. Skip if you cannot accept Go as the extension language and JS-via-`goja` (a pure-Go ES5.1 + partial ES6 runtime — no Node APIs, no `npm install`) as the scripting language; if you want to extend with Python / TS / native Node, choose Supabase Edge Functions, Appwrite, or roll your own.

## Vs already cataloged

- **Vs [`supabase-cli`](../supabase-cli/) / [`hasura-cli`](../hasura-cli/):** Supabase is Postgres-backed, much heavier, and assumes you run several services (db, auth, storage, realtime, edge functions). Hasura is a GraphQL gateway over Postgres / SQL Server. PocketBase is one binary + SQLite + REST/SSE — categorically smaller in scope and operationally one process to babysit. Pick PocketBase when "I need a backend on a small VPS today"; pick Supabase / Hasura when you have already chosen Postgres and want a managed-style experience around it.
- **Vs [`sqlite-utils`](../sqlite-utils/):** orthogonal. `sqlite-utils` is a great client/CLI for poking at any SQLite file, including PocketBase's `pb_data/data.db` for forensic queries, exports, and bulk imports.
- **Vs [`litestream`](../litestream/):** complementary. PocketBase makes the SQLite database; Litestream replicates it to S3-compatible storage so a host failure does not lose data. Most production PocketBase deployments run both.
- **Vs [`atlas`](../atlas/) / [`sqlc`](../sqlc/):** atlas is a schema-migration tool and sqlc generates typed query code; PocketBase has its own migration runner and an auto-generated REST API, so it does not need either when used as-is. If you `import` PocketBase as a Go library and want typed queries against custom tables, sqlc still pairs cleanly.
- **Vs [`caddy`](../caddy/) / [`frp`](../frp/):** PocketBase's built-in HTTPS via Let's Encrypt covers single-host TLS; for fleet TLS, multi-app routing, or tunneling to a NATed host, put Caddy or frp in front.

## Caveats

- **One process, one SQLite file.** That is the design. Vertical-scale only. Horizontal scale-out is explicitly out of scope; if you need it, you have outgrown PocketBase.
- **Backups are your job.** PocketBase ships an "auto-backup" feature in the admin UI that snapshots `pb_data/` to local files or S3 on a schedule, but verify your backups by restoring them on a fresh box. For PITR-style durability, run [`litestream`](../litestream/) alongside.
- **JS hooks run in `goja`, not Node.** No `require("fs")`, no `npm` packages with native bindings, no `fetch` from Node — PocketBase exposes its own `$http`, `$os`, `$filepath`, `$security` globals from the docs. If your hook genuinely needs Node ecosystem, write a Go extension instead and import the libraries you need.
- **Auth-token format is JWT.** Tokens are HS256-signed using a secret stored in `pb_data/`; rotating that secret invalidates every issued token. Treat `pb_data/` as a secret directory (chmod 700, exclude from world-readable backups).
- **Schema-as-code is opt-in.** The admin UI lets you click-create collections, which is great for prototyping but creates drift between environments. For anything beyond a prototype, write `pb_migrations/*.js` and disable "settings UI editing" in production.
- **0.x version.** PocketBase is pre-1.0 and the maintainer ships breaking changes between minor versions (collection schema model, hook API). Pin a version, read the changelog before upgrading, and keep a `pb_data/` backup before any version jump. v1.0 is on the public roadmap.
- MIT ([LICENSE.md](https://github.com/pocketbase/pocketbase/blob/master/LICENSE.md)) — permissive; safe to embed in proprietary products and ship as part of a closed-source binary.

## Example invocations

```bash
# Run as a standalone backend on http://127.0.0.1:8090
./pocketbase serve

# First-run admin: open http://127.0.0.1:8090/_/ and create the superuser

# Programmatic admin creation (CI / provisioning)
./pocketbase superuser create admin@example.com 'a-strong-password'

# Apply pending migrations from pb_migrations/
./pocketbase migrate up

# Generate a new migration template (Go or JS)
./pocketbase migrate create "add_status_to_posts"

# Snapshot pb_data/ on demand
./pocketbase admin backup create "pre-deploy-$(date +%Y%m%d)"

# Hit the auto-generated REST API (after creating a "posts" collection)
curl -s 'http://127.0.0.1:8090/api/collections/posts/records?perPage=5&sort=-created' | jq

# Auth as a user, then list owned records
TOKEN=$(curl -s -X POST http://127.0.0.1:8090/api/collections/users/auth-with-password \
          -H 'content-type: application/json' \
          -d '{"identity":"alice@example.com","password":"hunter2"}' | jq -r .token)

curl -s -H "Authorization: $TOKEN" \
     "http://127.0.0.1:8090/api/collections/posts/records?filter=owner='\$@request.auth.id'"

# Realtime: subscribe to all changes on the "posts" collection (SSE)
curl -N -H "Authorization: $TOKEN" \
     'http://127.0.0.1:8090/api/realtime'
```

## Why it fits the catalog

PocketBase represents the "single-binary BaaS" niche — a category distinct from the Postgres-stack BaaS world (Supabase / Hasura / Appwrite) and from raw embedded databases ([`sqlite-utils`](../sqlite-utils/), [`duckdb`](../duckdb/)). For agent / AI tooling specifically, it is a fast way to stand up a backend that an LLM-driven app can read/write without you provisioning a multi-service stack: collections become tool-callable REST endpoints, the SSE feed gives realtime context updates, and the file-storage API handles attachment / artifact persistence — all from one process the agent can also operate (create collections, run migrations) over its admin API.
