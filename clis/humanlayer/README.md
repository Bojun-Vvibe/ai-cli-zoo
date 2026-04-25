# humanlayer

> Snapshot date: 2026-04. Upstream: <https://github.com/humanlayer/humanlayer>
> Pinned versions: Python/TS SDK **v0.20.0**; `codelayer` desktop TUI
> **0.1.0-nightly-20260227** (nightly channel only).

A human-in-the-loop control plane for AI coding agents. Two halves:
(a) the `humanlayer` SDK (`hlyr` CLI + Python/TS clients) that lets a
running agent ask a human to approve specific tool calls or answer
specific questions over Slack / email / web, and (b) `codelayer`, a
local Tauri desktop app that orchestrates `claude-code`, `codex`,
`opencode` and `amp` as worker subagents on the same task, with a
shared session DB.

## 1. Install footprint

- SDK: `pip install humanlayer` or `npm i humanlayer`. Brings the
  `hlyr` CLI (Node).
- Daemon: `hld` is a Go binary that holds session state and brokers
  human-approval requests; one local process per laptop.
- TUI / desktop: `codelayer` ships as signed macOS / Linux nightly
  builds via the GitHub Releases page (`humanlayer/humanlayer`
  `nightly-*` tags) or `brew install --cask humanlayer/tap/codelayer`.
- Stores session state in `~/.humanlayer/` (SQLite via `y-schema.sql`).

## 2. License

Apache-2.0 (verified from `LICENSE` in the upstream repo; GitHub's
license API reports `NOASSERTION` because the file header is
`Apache Software License 2.0` rather than the SPDX-recognised exact
title — the body is verbatim Apache-2.0).

## 3. Models supported

Inherited from the underlying agent harness `codelayer` is driving:
`claude-code` (Anthropic Claude family), `codex` (OpenAI o-series /
GPT-5), `opencode` (any provider opencode supports — Anthropic, OpenAI,
Bedrock, OpenRouter, local), and `amp` (Sourcegraph Amp). The SDK
itself is model-agnostic — it sits on the *approval* hop, not the
inference hop, so any LLM provider works as long as the agent calls
`requestApproval()` / `humanAsTool()` before executing the gated tool.

## 4. MCP support

**Yes (server, indirectly).** The SDK exposes a `humanlayer-mcp` server
that any MCP-aware agent can mount; the server's tools (`request_approval`,
`request_human_input`, `human_as_tool`) become available as ordinary
MCP tools in the agent's surface. No first-party MCP client (humanlayer
isn't itself an LLM caller).

## 5. Sub-agent model

Manager / worker. `codelayer` is the manager process; each task spawns
one or more worker subagents (one `claude-code` process, one `codex`
process, ...) in their own working directories with their own session
state. The manager presents a unified diff + approval queue across all
of them. The SDK independently supports nested agents — any function
wrapped in `@require_approval()` becomes a gated tool the parent agent
can call.

## 6. Telemetry stance

**Off in OSS.** The local daemon and CLI do not phone home. If you
configure the hosted humanlayer.dev control plane (optional, used for
Slack / email approval routing when you don't want to self-host the
broker), that endpoint sees approval-request payloads. Self-host with
`HUMANLAYER_API_BASE=http://localhost:8080` to keep everything local.

## 7. Prompt-cache strategy

Inherited from the wrapped agent. `codelayer` does not insert itself
into the model call, so whatever Anthropic-ephemeral or OpenAI prefix
caching `claude-code` / `codex` / `opencode` already do continues to
apply unchanged.

## 8. Hot keybinds (TUI — `codelayer`)

| Key | Action |
|-----|--------|
| `n` | New task / session |
| `j` / `k` | Next / previous session in the queue |
| `Enter` | Open the highlighted session |
| `a` | Approve the pending tool call |
| `d` | Deny / edit the pending tool call |
| `f` | Open the diff viewer for the session's worktree |
| `t` | Switch worker model on the active session |
| `Ctrl+C` | Cancel running tool call |
| `q` | Back to the queue / quit |

## 9. Killer feature, weakness, when to choose

- **Killer:** **agent-agnostic approval queue with a real desktop
  inbox.** Any agent that calls the SDK's `request_approval()` lands
  in one place — Slack, email, the codelayer queue — and a single human
  can clear approvals for `claude-code`, `codex`, `opencode` and `amp`
  workers on parallel worktrees from one keyboard. The "fan out four
  agents on the same prompt, accept the winner" loop ships out of the
  box, with audit log per worker.
- **Weakness:** the desktop app is nightly-only at the time of
  snapshot — no stable `v1` tag for `codelayer` yet, and the SQLite
  schema is still moving. The SDK itself is stable (`v0.20.0`).
- **Choose it when:** you want to drive multiple coding agents in
  parallel against the same task with a human gate on every shell /
  write call, and you want the gate to be Slack-routable, not just a
  TTY prompt — the niche [container-use](../container-use/) handles at
  the sandbox layer, humanlayer handles at the human-decision layer.
