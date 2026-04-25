# composio

> Snapshot date: 2026-04. Upstream: <https://github.com/ComposioHQ/composio>

"**1000+ toolkits, tool search, context management, authentication,
and a sandboxed workbench to help you build AI agents that turn
intent into action.**" Composio is a hosted (and self-hostable)
tool-execution layer for agents: instead of every framework
re-implementing GitHub / Slack / Gmail / Jira / Notion / Linear /
Stripe / GoogleDrive integrations and re-solving OAuth, Composio
exposes a single SDK that returns ready-to-call tool schemas to
any agent runtime (LangChain, LangGraph, OpenAI Agents SDK,
CrewAI, AutoGen, LlamaIndex, Pydantic-AI, Vercel AI SDK), handles
the OAuth dance per end-user, and proxies the actual API calls
through a sandboxed runtime that returns structured outputs the
LLM can consume on the next step.

## 1. Install footprint

- `npm install @composio/cli` (or `pnpm`/`yarn`) installs the CLI
  binary; the package resolves to a single `composio` entry point.
- Language SDKs: `pip install composio` (Python), `npm install
  @composio/core` plus framework adapters (`@composio/openai`,
  `@composio/langchain`, `@composio/vercel`, …).
- `composio login` walks through cloud-account creation; for
  fully self-hosted use, point `COMPOSIO_BASE_URL` at a private
  deployment (Docker images and Helm charts in the repo's
  `deploy/` tree).
- Node ≥ 20 / Python ≥ 3.10. Mac / Linux / Windows; agents and
  tool execution can run on the user's machine, in Composio
  Cloud, or in a dedicated sandbox tier.

## 2. Repo + version + license

- Repo: <https://github.com/ComposioHQ/composio>
- Latest release: **@composio/cli@0.2.26** (2026-04-25)
- License: **MIT** —
  <https://github.com/ComposioHQ/composio/blob/next/LICENSE>
- HEAD SHA: `69f9ee998009610b8d67d65d3b592702d6b0dda7`
- Default branch: `next`
- Language: TypeScript (Python SDK alongside)

## 3. Models supported

Composio is **model-agnostic** — it returns tool schemas in the
shape your chosen agent runtime expects (OpenAI tool-calling
JSON, Anthropic tool blocks, Gemini function declarations,
LangChain `StructuredTool`, CrewAI `BaseTool`, Vercel AI SDK
`tool()`), so any model your runtime drives can call the
toolkits. The CLI does not bundle an LLM call itself; it brokers
between agent runtime and external SaaS APIs.

## 4. Simple usage

```bash
npm install -g @composio/cli
composio login                       # browser OAuth into Composio
composio toolkits                    # list all 1000+ toolkits
composio toolkits add github         # add GitHub toolkit + OAuth
composio mcp create my-stack \
  --toolkits github,slack,linear \
  --transport http                   # spin up a hosted MCP server
```

```python
# Python: feed Composio tools into the OpenAI Agents SDK
from composio import Composio
from composio_openai import OpenAIProvider

composio = Composio(provider=OpenAIProvider())
tools = composio.tools.get(
    user_id="alice@example.com",
    toolkits=["GITHUB", "LINEAR", "SLACK"],
)

from openai import OpenAI
client = OpenAI()
resp = client.responses.create(
    model="gpt-4.1",
    input="Open a Linear issue for the failing CI run on the main branch",
    tools=tools,
)
composio.provider.handle_tool_calls(resp, user_id="alice@example.com")
```

## 5. Why it's interesting

- **OAuth as a service for agents** — per-end-user token
  storage, refresh, and scope management for 1000+ SaaS APIs,
  so a multi-tenant agent product does not have to build a
  separate "Connect your Slack" flow per integration.
- **One toolkit definition, many runtimes** — the same
  `GITHUB_CREATE_ISSUE` action surfaces as an OpenAI tool, a
  LangChain `StructuredTool`, a CrewAI `BaseTool`, a Vercel AI
  SDK `tool()`, and an MCP tool, with consistent input/output
  schemas; framework hops do not require rewriting the tool
  layer.
- **MCP server generation built in** — `composio mcp create`
  packages selected toolkits as a hosted MCP endpoint that any
  MCP-aware agent ([`opencode`](../opencode/) /
  [`claude-code`](../claude-code/) / [`crush`](../crush/) /
  [`fast-agent`](../fast-agent/) / [`pydantic-ai`](../pydantic-ai/))
  can mount with a single URL.
- **Triggers + Actions** — toolkits expose both pull-style
  actions (call this API) and push-style triggers (Slack message,
  GitHub PR opened, calendar event), so event-driven agents
  ("when a new PR is opened, run a code-review pipeline") use
  the same SDK as request/response agents.
- **Sandboxed execution** — tool calls run in a per-user
  sandbox with rate limiting and audit logging; bad LLM output
  cannot exfiltrate by hitting `evil.com` from your laptop.

## 6. Caveats

- The hosted control plane is the default surface; fully
  air-gapped deployments require running the self-hosted images
  (`deploy/`) plus your own Postgres + Redis, which is heavier
  than dropping a tools file into your agent.
- 1000+ toolkits is a long tail — coverage of the top SaaS APIs
  (GitHub, Slack, Linear, Notion, Gmail, GoogleDrive, Jira,
  HubSpot, Stripe) is excellent; obscure verticals can lag the
  underlying API by a release.
- Pricing meters tool calls in the hosted tier; high-volume
  agentic workloads should benchmark against a self-hosted
  install before committing.
- The `next` branch is the default — long-lived forks against
  `main` will drift; PRs go into `next`.
