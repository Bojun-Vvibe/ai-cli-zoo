# mcpo

> Snapshot date: 2026-04. Upstream: <https://github.com/open-webui/mcpo>

A **simple, secure MCP-to-OpenAPI proxy server** that takes any
Model Context Protocol server (stdio, SSE, or Streamable HTTP)
and exposes its tools as a standard OpenAPI / REST endpoint with
auto-generated `/docs` Swagger UI, bearer-token auth, and
per-tool path routing. Built by the Open WebUI team to bridge
"the MCP server I want to use" with "the LLM client / chat UI /
automation tool that only speaks REST", mcpo turns any MCP
server into a normal HTTP API in one command.

It is the catalog's reference for **giving an MCP server an
OpenAPI face** so it can be consumed by everything that *isn't*
yet MCP-aware (curl, Postman, OpenAI function-calling clients,
n8n, Zapier, plain Python `requests`, web frontends).

## 1. Install footprint

- `uvx mcpo --port 8000 -- uvx mcp-server-time --local-timezone=America/New_York`
  is the canonical run-it-without-installing path.
- `pip install mcpo` for a persistent install (Python 3.10+).
- Docker image: `ghcr.io/open-webui/mcpo:latest`.
- ~20 transitive deps: `fastapi`, `uvicorn`, `mcp`, `httpx`,
  `pydantic`, `click`, `passlib`. ~35 MB venv.
- Models: none — mcpo is a transport adapter, not an LLM
  client.
- Backends: any MCP server (Python via `uvx`, Node via `npx`,
  Go / Rust binaries, Docker images, or remote SSE / HTTP
  URLs).

## 2. Repo, version, license

- Repo: <https://github.com/open-webui/mcpo>
- Version checked: **v0.0.20** (latest tagged release; the
  project is in active 0.x but the surface is stable).
- HEAD pinned at this snapshot:
  `788ff92e5288a899a743a252edd5748f4ad4ab1f`.
- License: MIT. License file at
  [`LICENSE`](https://github.com/open-webui/mcpo/blob/main/LICENSE).

## 3. What it actually does

Single-server mode wraps one MCP server:

```bash
uvx mcpo --port 8000 --api-key "secret" -- \
  uvx mcp-server-time --local-timezone=America/New_York
```

mcpo spawns the MCP server as a subprocess, speaks MCP to it
over stdio, introspects its `tools/list`, and generates a
FastAPI app where each tool becomes a `POST /<tool_name>`
endpoint with the tool's input schema as the request body and
its output schema as the response. `GET /docs` renders Swagger
UI; `GET /openapi.json` returns the spec.

Multi-server mode loads a config file:

```json
{
  "mcpServers": {
    "memory":     { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-memory"] },
    "time":       { "command": "uvx", "args": ["mcp-server-time"] },
    "filesystem": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"] }
  }
}
```

`mcpo --config config.json --port 8000` mounts each server
under its own path prefix (`/memory/*`, `/time/*`,
`/filesystem/*`) with per-server `/docs`. Remote MCP servers
(SSE / Streamable HTTP) are mounted by URL via the same config
schema using `"type": "sse"` or `"type": "streamable_http"`.

What mcpo gives you on top of the raw MCP server:

1. **Auto-generated Swagger / OpenAPI** at `/docs` per server.
2. **Bearer-token auth** (`--api-key`) gating every endpoint;
   no auth in the underlying MCP server is required.
3. **CORS + standard FastAPI middleware** so browsers can call
   it directly.
4. **Hot reload** of the config file (`--hot-reload`) for
   adding / removing servers without process restart.
5. **One Docker image** that hosts a fleet of MCP servers
   behind one HTTP port — the deployable shape Open WebUI uses
   in production.

## 4. MCP support

mcpo speaks MCP on the *backend* side (it is an MCP client
that proxies a server) and OpenAPI on the *frontend* side.
Its raison d'être is exactly the MCP-to-non-MCP bridge: it
does not expose an MCP endpoint of its own. If you need the
inverse (OpenAPI / REST → MCP server) the Prefect
[`fastmcp`](../fastmcp/) `FastMCP.from_openapi(...)` constructor
is the canonical answer.

## 5. Sub-agent model

None — mcpo is a stateless reverse proxy. Each HTTP request
opens a tool call against the upstream MCP session that the
server's stdio transport keeps alive; there is no agent loop,
no planning, no fan-out. Concurrency comes from FastAPI /
Uvicorn's async workers; one mcpo process can multiplex many
HTTP clients onto one MCP backend session.

## 6. Telemetry stance

Off by default. mcpo emits no analytics and makes no outbound
calls beyond what the wrapped MCP servers do. Standard FastAPI
access logs go to stdout; OpenTelemetry can be added with the
usual `opentelemetry-instrument` wrapper since the underlying
stack is plain FastAPI + httpx.

## 7. Token / context strategy

Not applicable — mcpo doesn't see prompts or contexts; it
relays JSON tool-call payloads. Response-size limits are
inherited from the FastAPI / Uvicorn defaults (configurable
via standard FastAPI knobs) and from the upstream MCP server's
own truncation behavior.

## 8. Hot keybinds

None — mcpo is a long-running HTTP server, not a TUI. The
interactive surface is `/docs` (Swagger UI) plus whatever HTTP
client (curl, Postman, browser, n8n node) consumes the
generated REST API.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **One-command MCP-to-OpenAPI bridge** with
auto-generated Swagger UI, bearer auth, and multi-server
mounting via a Claude-Desktop-shaped config file — the smallest
amount of glue between "the MCP ecosystem" and "every HTTP
client on earth". Particularly valuable for chat UIs (Open
WebUI, LibreChat) and automation platforms (n8n, Zapier, Make)
that have OpenAPI-tool support but no MCP support yet, and for
giving curl-level debuggability to MCP servers whose only
client is otherwise Claude Desktop.

**Weakness.** The translation is one-way (MCP → OpenAPI only);
MCP-specific features that don't have OpenAPI analogs
(progress notifications, sampling, elicitation, roots) are
either dropped or simulated as polling endpoints. Each mcpo
process owns one stdio MCP subprocess per server — high-traffic
deployments need to either pre-spin multiple replicas or
prefer MCP servers that support concurrent sessions natively.
Auth is bearer-token only; OAuth / mTLS / per-tool RBAC need an
upstream gateway.

**Choose mcpo when** you have an MCP server you want to consume
from a non-MCP client (chat UI, low-code automation, OpenAI
function-calling client, browser frontend), when your team
already standardizes on OpenAPI / Swagger for tool documentation
and you want MCP servers to fit the same review workflow, or
when you need a single HTTP entrypoint that fans out to a fleet
of MCP servers behind one API key. **Choose something else
when** the consumer already speaks MCP natively (point it at
the MCP server directly), when you need OpenAPI → MCP (use
[`fastmcp`](../fastmcp/)'s `from_openapi`), or when you need
heavy auth / rate-limit / observability and an API gateway
(Kong, Tyk, APISIX) in front of the MCP server is a better fit.
