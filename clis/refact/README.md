# refact

> Snapshot date: 2026-04. Upstream: <https://github.com/smallcloudai/refact>

"**AI Agent that handles engineering tasks end-to-end: integrates with
developers' tools, plans, executes, and iterates until it achieves a
successful result.**" Refact ships as a self-hostable **agent server**
(Rust core) plus IDE plugins (VS Code, JetBrains) and a CLI / REST
surface. The product covers code completion, chat, and an autonomous
agent loop that can call shell, run tests, browse a database, and open
PRs — all behind a server you can run on your own GPU box or against
hosted endpoints.

## 1. Install footprint

- Self-host (recommended): Docker image
  `smallcloud/refact_self_hosting:latest`, or build the `refact-lsp`
  Rust binary from source. Brings up an HTTP + LSP server that the IDE
  plugins and CLI talk to.
- IDE-side: `Refact.ai` extension in VS Code Marketplace / JetBrains
  Marketplace; both point at a configurable server URL (hosted or
  self-hosted).
- CLI surface: `refact-lsp` is the daemon; agent invocation goes through
  the chat / agent endpoints (`POST /v1/chat`, `POST /v1/agent`) or the
  IDE plugin's "Agent" panel.
- GPU optional: server can run CPU-only against external LLM endpoints,
  or host local fine-tuned completion models on GPU for the inline
  completion side.

## 2. Repo + version + license

- Repo: <https://github.com/smallcloudai/refact>
- Latest release: **server/v1.11.2**
- License: **BSD-3-Clause** —
  <https://github.com/smallcloudai/refact/blob/main/LICENSE>
- Default branch: `main`
- Language: Rust (core daemon) + Python (training / fine-tune side) +
  TS (IDE plugins)

## 3. Models supported

Two layers. **Completion** (inline, low-latency): bundled Refact-1.6B
and StarCoder-family models served from the self-hosted GPU box, or
swap in any HF causal-LM checkpoint. **Chat / agent** (planning + tool
use): OpenAI, Anthropic, Groq, DeepSeek, any OpenAI-compatible endpoint
(Ollama, vLLM, LiteLLM, OpenRouter), plus the hosted Refact endpoint.
Routing is per-feature in `bring-your-own-key.yaml`.

## 4. MCP support

Yes, both sides. The agent loop can mount **MCP servers as tools**
(declared in `~/.config/refact/integrations.d/mcp_*.yaml`), and
`refact-lsp` itself can be exposed as an MCP server (`refact-lsp
--mcp-server`) so other catalog agents
([`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`crush`](../crush/)) can call its `definition` / `references` /
`patch` / `tree` tools.

## 5. Sub-agent model

Single agent loop per task, but with first-class **integrations** as
tools: `cmdline`, `pdb` debugger, `postgres`, `mysql`, `chrome`
(headless browser), `github`, `gitlab`, `jira`, `slack`,
`docker`, plus arbitrary MCP servers. The agent plans, picks an
integration, runs it, observes output, and loops. No automatic
sub-agent spawn — composition is via integrations, not nested agents.

## 6. Telemetry stance

**On by default in the OSS server, opt-out via config.** The bundled
self-hosting setup posts anonymized completion telemetry to the
`smallcloud` endpoint to drive the inline-completion model fine-tune
flywheel; turn off via `telemetry: false` in the server config or by
setting `TELEMETRY_BASIC_DEST=` empty. Hosted plan obviously sees
prompts.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Self-hostable end-to-end agent stack with a
real integrations catalog.** Where most catalog agents are CLIs that
shell out to whatever happens to be on `$PATH`, Refact ships a
declarative integrations layer — one YAML to wire up `postgres` with
read-only creds, one YAML to wire up `chrome` for browser steps, one
YAML to mount a GitHub PAT — and the agent loop picks and chains them
during a task. The whole thing runs behind your VPC if you want, with
the LSP daemon, completion model, chat model routing, and PR-opening
flow all in one Docker image. Few alternatives let you keep the agent
loop on-prem while still reaching cloud LLMs for the chat tier.

**Weakness.** Heavy. The server is a real piece of infra (Rust daemon
+ Python fine-tune side + Postgres for state), the integrations layer
is YAML-heavy enough that you will spend an afternoon on the first
setup, and the IDE plugins are the primary UX — the pure-CLI surface
is thinner than [`aider`](../aider/) or [`opencode`](../opencode/).
Telemetry-on-by-default in the OSS build is a footgun for regulated
environments; flip it off explicitly. Documentation lags the code on
the agent / MCP side.

**When to choose.** You want **inline completion + chat + autonomous
agent + PR opening** under one roof, you want it self-hostable, and
you are willing to run a real server (GPU optional but recommended for
the completion model). Pair with [`continue`](../continue/) or
[`cline`](../cline/) if you only want the IDE side, with
[`opencode`](../opencode/) / [`claude-code`](../claude-code/) if you
want a terminal-first agent and not a server, or with
[`tabby`](https://github.com/TabbyML/tabby) if your need is *only*
self-hosted completion. Skip if you have no infra appetite or if
"agent" for you means a single `aider` REPL.
