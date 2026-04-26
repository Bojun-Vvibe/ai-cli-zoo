# tracecat

> Snapshot date: 2026-04-26. Upstream: <https://github.com/TracecatHQ/tracecat>

"**Open-source security automation platform for teams and AI agents.**"
Tracecat is a Tines / Splunk-SOAR alternative built around a YAML
workflow DSL and a Temporal execution engine. The 1.x line treats AI
agents as first-class actors: a workflow can call an LLM as a step,
and an agent can in turn call deterministic Tracecat actions
(SIEM lookups, EDR isolations, ticketing, IAM revokes) as
tool-calls. The CLI surface is the `tracecat` Python package plus the
hosted / self-hosted FastAPI + Next.js stack you boot via Docker
Compose; agent-mode integrations expose Tracecat actions as
OpenAI-shape tools and as MCP tools for any MCP-aware client.

## 1. Install footprint

- Self-host: `git clone https://github.com/TracecatHQ/tracecat &&
  ./env.sh && docker compose up -d` (Postgres + Temporal + API +
  worker + UI).
- Python SDK / CLI: `pip install tracecat` for action authoring +
  workflow `tracecat run` invocations against a running deployment.
- 100+ pre-built integrations: AWS / GCP / Azure security APIs,
  CrowdStrike, SentinelOne, Wiz, Snyk, Okta, Auth0, Slack, Jira,
  ServiceNow, VirusTotal, GreyNoise, AbuseIPDB, urlscan, MISP,
  TheHive, S1QL / KQL / SPL search adapters, plus a generic
  `core.http_request` action.

## 2. Repo + version + license

- Repo: <https://github.com/TracecatHQ/tracecat>
- Latest release: **1.0.0-beta.42** (2026-04-21)
- License: **AGPL-3.0** —
  <https://github.com/TracecatHQ/tracecat/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (backend, SDK, agent layer) + TypeScript (Next.js
  console)
- Stars: ~3.6k

## 3. Models supported

Provider-agnostic on the agent step. The bundled `core.ai.action`
calls a configured LLM (OpenAI, Anthropic, Gemini, Bedrock,
LiteLLM-routed long tail) with the workflow context plus the
catalog of action schemas the agent is allowed to invoke. Tool calls
returned by the model are dispatched as normal Tracecat actions —
the same audit log, retry policy, and approval gate apply whether a
human or an LLM triggered them.

## 4. Notable angle

**Deterministic SOAR engine + LLM as a constrained tool-caller, not
the controller.** Tracecat workflows are durable Temporal workflows
with replay, time-travel debugging, and signed audit log; an
`agent` step is just one node in that DAG, scoped to a whitelisted
subset of actions, with optional human-approval gates between
proposal and execution. This inverts the usual agent-framework
shape (LLM is the loop, tools are leaves) — here the workflow is the
loop, the LLM is one leaf, and a security team can ship an agent
playbook through the same review process as any other automation,
with the same blast-radius controls.

## 5. Last verified

2026-04-26 via `gh api repos/TracecatHQ/tracecat`.
