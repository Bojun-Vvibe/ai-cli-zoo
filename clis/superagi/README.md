# superagi

> Snapshot date: 2026-04. Upstream: <https://github.com/TransformerOptimus/SuperAGI>
> Pinned release: `v0.0.14`. License file: `LICENSE` (sha `1ec41bdb6ecfdc2db7d578233d3a603ab51d2aad`).

A dev-first autonomous-agent framework that ships with a web GUI, a CLI
runner, and a Postgres-backed agent registry. Older than most of the
"agent platform" cohort and still one of the few that treats a long-lived
agent as a first-class persisted object, not a chat session.

## 1. Install footprint

- `git clone https://github.com/TransformerOptimus/SuperAGI && docker-compose up -d` is the supported path.
- Bare-metal install requires Python 3.10+, Postgres 14+, Redis, and a
  `config.yaml`. The Docker compose bundle wires all four for you.
- ~2.5 GB of images; the React GUI is its own container on port 3000.
- No real "single binary" — this is a server with a CLI client (`superagi`).

## 2. License

MIT.

## 3. Models supported

OpenAI (GPT-4 family), Anthropic (Claude), Google PaLM/Gemini, local via
Ollama and LM Studio, plus any OpenAI-compatible endpoint configured per
agent. Per-agent model selection is stored in the database, not a global
flag.

## 4. MCP support

**No.** Tool integrations use SuperAGI's own "toolkit" interface (Python
classes registered in the DB). There are 30+ first-party toolkits (Jira,
Slack, Google Calendar, GitHub, web-search, file IO, image gen) but none
speak MCP. Bridging would require a custom toolkit wrapper.

## 5. Sub-agent model

Yes — explicitly. An agent can spawn a child agent via the
`AgentExecutor` toolkit and wait on its output. The parent's iteration
loop blocks on the child's terminal state. Child runs are recorded in
the same Postgres table as top-level runs, so you can audit a tree of
delegations after the fact.

## 6. Telemetry stance

No outbound telemetry from the framework itself. The OpenAI/Anthropic
keys you configure see prompts; nothing else does. Optional Sentry hook
is opt-in via env var.

## 7. Prompt-cache strategy

None built in. Each agent step is a fresh API call with the full
constructed prompt. On long-running agents this is the primary cost
driver — pair with a provider that does automatic prefix caching
(Anthropic, OpenAI) to claw some of it back.

## 8. Hot keybinds (CLI)

The shipped CLI is a thin client over the REST API. Most interaction is
via the web GUI on `:3000`; the CLI verbs are:

| Command | Action |
|---------|--------|
| `superagi agent create` | Create an agent definition |
| `superagi agent run <id>` | Kick off an execution |
| `superagi agent status <run-id>` | Poll a running execution |
| `superagi toolkit list` | List installed toolkits |
| `superagi toolkit install <name>` | Install a toolkit from the registry |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **persisted agent registry**. Every agent is a row in
  Postgres with goals, tools, model config, and a full execution
  history. You can stop a long-running agent, restart it days later,
  and inspect its full reasoning trace. Most "agent CLIs" forget
  everything when the process exits.
- **Weakness:** heavyweight to operate. Three containers minimum, a
  schema-migrated database, and a React GUI you can't easily skip.
  Overkill for "I want to summarize a PDF."
- **Choose it when:** you're building a small fleet of background
  agents (research bots, monitors, scheduled scrapers) that need to
  outlive any single shell session and produce auditable run history.
