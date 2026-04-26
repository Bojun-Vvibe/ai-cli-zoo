# fastmcp

> Snapshot date: 2026-04. Upstream: <https://github.com/PrefectHQ/fastmcp>

A **Pythonic framework for building Model Context Protocol (MCP)
servers and clients** that hides the JSON-RPC plumbing behind
decorator-driven `@mcp.tool`, `@mcp.resource`, and `@mcp.prompt`
primitives. Started by Jeremiah Lowin as an indie project, now
stewarded by Prefect, fastmcp 2.x is the reference high-level
implementation that the official `mcp` Python SDK now embeds at
its core — which means most production MCP servers in the wild
are either fastmcp or a thin wrapper around it.

It is the catalog's reference for **writing your own MCP server
in ~20 lines of Python and having it work in Claude Desktop /
opencode / cline / continue / goose without further glue**.

## 1. Install footprint

- `pip install fastmcp` (Python 3.10+).
- `uv tool install fastmcp` for the CLI.
- ~25 transitive deps (httpx, pydantic, mcp, anyio, starlette,
  uvicorn, click). ~40 MB venv.
- Models: none — fastmcp is the *server* / *client* substrate
  that LLMs talk to via MCP. It does not call any model itself.
- Transports: stdio (default for desktop hosts), Streamable HTTP
  (modern remote), SSE (legacy remote), in-memory (testing).

## 2. Repo, version, license

- Repo: <https://github.com/PrefectHQ/fastmcp>
- Version checked: **v3.2.4** (latest tagged release in the
  v3.x line; v2 is still maintained but v3 is the current
  default for new projects).
- HEAD pinned at this snapshot:
  `d0315974fa844b2424c93f4ad43c8f2d543ff51a`.
- License: Apache-2.0. License file at
  [`LICENSE`](https://github.com/PrefectHQ/fastmcp/blob/main/LICENSE).

## 3. What it actually does

A server is one decorated module:

```python
from fastmcp import FastMCP

mcp = FastMCP("weather")

@mcp.tool
def get_forecast(city: str, days: int = 3) -> dict:
    """Return an N-day forecast for a city."""
    return fetch_forecast(city, days)

@mcp.resource("weather://stations")
def list_stations() -> list[str]:
    return load_station_index()

if __name__ == "__main__":
    mcp.run()  # stdio by default
```

Run it: `fastmcp run weather.py`, or wire it into Claude Desktop
with `fastmcp install weather.py`, or expose it over HTTP with
`mcp.run(transport="http", port=8000)`.

What fastmcp does for you under the decorators:

1. **Schema generation** — type hints + docstrings become MCP
   tool / resource / prompt JSON schemas; `pydantic` models in
   signatures become nested schemas; `Field(...)` annotations
   propagate descriptions and constraints.
2. **Transport multiplexing** — same server runs over stdio,
   Streamable HTTP, SSE, or in-memory without code changes.
3. **Composition** — `mcp.mount(other_mcp, prefix="weather")`
   merges multiple servers; `FastMCP.from_openapi(spec_url)` and
   `FastMCP.from_fastapi(app)` auto-generate a server from an
   existing OpenAPI / FastAPI surface.
4. **Client side** — `Client(server_url_or_path)` is the
   counterpart for writing custom MCP clients (Python agents
   that consume MCP servers), with sampling, elicitation,
   roots, progress, and cancellation all wired through.
5. **Auth** — pluggable `BearerAuthProvider` /
   `OAuthProvider` for HTTP transports; per-tool permission
   gates via `@mcp.tool(annotations={...})`.

## 4. MCP support

This *is* the MCP support. fastmcp implements both server and
client sides of the protocol (2024-11-05 + 2025-03-26 + later
spec revisions), and the official `mcp` Python SDK now bundles
fastmcp's server core internally — so the high-level surface
many other catalog tools use (e.g. servers shipped by
[`composio`](../composio/) integrations,
[`mcp-agent`](../mcp-agent/) sub-servers,
[`pydantic-ai`](../pydantic-ai/)'s tool-server bridge) is fastmcp
under the hood.

## 5. Sub-agent model

Not an agent framework — fastmcp is the protocol layer
*beneath* agents. There is no LLM loop, no planner / executor,
no spawn semantics. Composition is server-mount + proxy:
`FastMCP.as_proxy(remote_url)` wraps a remote server as a local
one (useful for stdio-only hosts like Claude Desktop that need
HTTP MCPs reflected through stdio).

## 6. Telemetry stance

Off by default. The framework itself emits no analytics; the
optional `fastmcp` CLI's `install` / `dev` / `inspect` commands
talk only to localhost. OpenTelemetry instrumentation is opt-in
via standard Python OTel auto-instrumentation against the
`starlette` / `httpx` layers fastmcp is built on. Egress in a
deployed server = whatever HTTP / DB / SDK calls your `@mcp.tool`
implementations make.

## 7. Token / context strategy

Not applicable — fastmcp doesn't talk to LLMs. The server-side
concerns it does have are **payload size** (large
`@mcp.resource` returns blow up the host's context budget;
fastmcp emits `Resource.size` metadata so clients can
preflight) and **tool-result truncation** (`text` results
should typically stay under ~25 KB for most hosts; fastmcp
exposes streaming `Context.report_progress` / `Context.info`
so long-running tools push partial output instead of one giant
final string).

## 8. Hot keybinds

None — fastmcp is a Python library + a thin Click CLI
(`fastmcp run`, `fastmcp dev`, `fastmcp install`, `fastmcp
inspect`). The interactive surface is whatever MCP host
(Claude Desktop, opencode, cline, continue, goose) consumes
the server.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Decorator-to-MCP-server in twenty lines**
with auto-generated schemas, transport portability (stdio /
HTTP / SSE / in-memory) on a flag, server composition via
`mount` and `as_proxy`, and one-shot generation from existing
OpenAPI / FastAPI surfaces — and because the official Python
MCP SDK embeds fastmcp's server core, what you write is what
the rest of the ecosystem already speaks. The included
`Client` class makes fastmcp the only first-class library for
writing *custom MCP clients* in Python (most other libraries
are server-only).

**Weakness.** The MCP spec itself is still evolving (auth,
elicitation, sampling semantics shifted twice in 2025) and
fastmcp tracks the spec aggressively, so v2 → v3 had real
breaking changes around transport defaults and auth providers;
servers pinned to `fastmcp<3` need a small migration. The
high-level decorators hide protocol details that occasionally
matter (custom JSON-RPC error codes, partial-result
streaming semantics) and force you to drop down to the lower
`mcp` SDK. Authentication for HTTP transports works but is
less battle-tested than the stdio path most desktop hosts use.

**Choose fastmcp when** you are writing an MCP server (or
client) in Python and want decorators + type-hint-driven
schemas + transport portability without hand-rolling JSON-RPC,
when you need to compose / mount multiple MCP servers behind a
single endpoint, or when an existing OpenAPI / FastAPI service
needs an MCP face bolted on. **Choose something else when**
you are writing the server in TypeScript / Go / Rust (use the
respective official MCP SDKs), when you only need to *consume*
MCP servers from an agent framework (most catalog agents
already speak MCP — e.g. [`pydantic-ai`](../pydantic-ai/),
[`mcp-agent`](../mcp-agent/), [`goose`](../goose/),
[`opencode`](../opencode/)), or when the protocol layer you
actually need is tool-calling without MCP and a plain Python
function registry suffices.
