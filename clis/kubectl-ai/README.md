# kubectl-ai

> Snapshot date: 2026-04. Upstream: <https://github.com/GoogleCloudPlatform/kubectl-ai>
> Binary name: `kubectl-ai` (also installable as the `kubectl ai` plugin via Krew)

`kubectl-ai` is the **Kubernetes-native chat agent** of the catalog.
Where [`k8sgpt`](../k8sgpt/) runs a deterministic analyzer sweep and
narrates the findings, `kubectl-ai` is the inverse shape: an
agentic REPL that holds a conversation, plans which `kubectl`
commands to run, executes them against the cluster your `KUBECONFIG`
points at, and feeds the output back into its own next turn. It is
the closest thing in the catalog to "[`aider`](../aider/) but for
your live cluster instead of your repo".

The intended loops:

1. `kubectl-ai` — drop into an interactive REPL with the Kubernetes
   toolset wired up; ask "why is the nginx deployment unhealthy?",
   the agent runs `kubectl get` / `describe` / `logs` and answers.
2. `kubectl-ai "scale frontend to 5 replicas"` — one-shot mode;
   the agent plans, asks for confirmation on the mutating
   `kubectl scale` call, executes, reports.
3. `kubectl ai ...` (as a kubectl plugin via Krew) — same surface
   from inside the kubectl ecosystem, with the binary name
   `kubectl-ai` discovered on PATH.

## 1. Install footprint

- Single static Go binary, no runtime deps.
- Install paths:
  - `curl -sSL https://raw.githubusercontent.com/GoogleCloudPlatform/kubectl-ai/main/install.sh | bash`
    (Linux / macOS one-liner; downloads the latest release tarball
    and drops the binary in `/usr/local/bin/`).
  - `kubectl krew install ai` — installs as a kubectl plugin so you
    can invoke `kubectl ai ...`.
  - Manual: download a release tarball from the GitHub releases
    page for your OS/arch, untar, `chmod +x`, move into PATH.
  - `nix-shell -p kubectl-ai` for one-shot use on NixOS.
- Reads the same `~/.kube/config` your `kubectl` already uses; no
  separate auth wiring against the cluster.
- LLM credentials configured via env vars (`GEMINI_API_KEY`,
  `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.) or via a config file
  at `~/.config/kubectl-ai/config.yaml`. Default provider is
  Gemini; switch with `--llm-provider openai` /
  `--model gpt-4o` etc.
- Docker image: `ghcr.io/googlecloudplatform/kubectl-ai`.

## 2. Repo, version, license

- Repo: <https://github.com/GoogleCloudPlatform/kubectl-ai>
- Version: **v0.0.31** (2026-03).
- License: Apache-2.0. License file at the repo root:
  [`LICENSE`](https://github.com/GoogleCloudPlatform/kubectl-ai/blob/main/LICENSE).
- Language: Go.

## 3. Models supported

Provider matrix (selected via `--llm-provider`):

- **Gemini** (default; `gemini-2.0-flash`, `gemini-2.5-pro`, etc.)
- **OpenAI** and any **OpenAI-compatible** endpoint
  (Azure OpenAI, Groq, OpenRouter, vLLM, LM Studio, LiteLLM)
- **Anthropic** Claude (Sonnet / Opus / Haiku)
- **Vertex AI** (Gemini through Vertex)
- **Ollama** for fully local operation (`--llm-provider ollama
  --model llama3.1`)
- **llama.cpp** server (`--llm-provider llama.cpp`)

Function-calling / tool-use is the spine — every supported provider
must speak structured tool calls; the agent loop will not work
against a chat-only endpoint.

## 4. MCP support

Both directions, as of v0.0.31:

- **MCP client.** `kubectl-ai` can consume MCP servers declared in
  its config; the tools they expose join the built-in `kubectl` /
  `bash` toolset and become callable inside the agent loop. This is
  how you bolt on, e.g., a Prometheus or Grafana MCP server for
  metrics-aware investigations.
- **MCP server.** `kubectl-ai --mcp-server` exposes the agent's
  cluster-aware toolset over stdio for an MCP-aware front-end
  (`opencode`, `claude-code`, `cline`, `crush`, `codex`, `goose`,
  `continue`) to drive. The MCP client gets `kubectl_get`,
  `kubectl_describe`, `kubectl_logs`, `kubectl_apply`,
  `bash_command`, etc., as discrete tools without having to
  re-implement the safety prompts.

## 5. Sub-agent model

None as a first-class concept. There is one agent loop per
invocation; tool calls are sequential (or parallel where the
provider supports it). What looks like sub-agency in practice is
the MCP-server boundary: a downstream MCP client treats
`kubectl-ai` as a single tool surface and is itself the sub-agent
controller.

## 6. Telemetry stance

**Off.** No analytics ping in the binary. Network egress is
exactly: the Kubernetes API server (read by default; writes only
on confirmed mutating commands), the configured LLM provider, and
any MCP servers you wire up. Air-gapped operation is supported via
Ollama or llama.cpp as the LLM provider.

## 7. Token / context strategy

- Stateful within a REPL session: the conversation transcript
  (system prompt + tool calls + tool outputs + user turns) is
  resent each turn, so context grows linearly with the
  investigation length.
- Tool outputs are *not* truncated by default — a `kubectl logs`
  on a chatty pod can blow the budget. Use `--max-tokens N` /
  `--max-iterations N` / explicit `kubectl logs --tail=200` in
  prompts to cap.
- One-shot mode (`kubectl-ai "..."`) starts with a fresh context
  every time, which is the right default for CI / scripts.
- No prefix caching layer in-process; cache hits depend on the
  underlying provider (Anthropic prompt caching, Gemini implicit
  caching, etc.) and the stability of your system prompt.

## 8. Hot keybinds

REPL is line-oriented (no full-screen TUI). Useful one-shot shapes
and slash-commands:

- `kubectl-ai` — start an interactive session.
- `kubectl-ai "<question>"` — one-shot, exits when the agent
  reports.
- `kubectl-ai --quiet "<question>"` — suppress thinking, print
  only the final answer (script-friendly).
- `kubectl-ai --kubeconfig /path` — point at a non-default kubeconfig.
- `kubectl-ai --llm-provider openai --model gpt-4o "..."` —
  override provider/model per call.
- `kubectl-ai --mcp-server` — run as an MCP server over stdio.
- Inside the REPL: `exit` / `quit` to leave; `clear` to reset
  context; mutating tool calls prompt for `y/n` before execution.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Native dual MCP role for cluster work.**
`kubectl-ai` is the only entry in the catalog that is both an
agent CLI *and* a cluster-aware MCP server, with first-party
maintenance from a major cloud platform team. This means you can:

- Use it directly when you want a chat-shaped Kubernetes
  investigation tool.
- *Or* mount it as `kubectl-ai --mcp-server` inside
  [`opencode`](../opencode/) / [`claude-code`](../claude-code/) /
  [`crush`](../crush/) / [`continue`](../continue/) and let those
  agents inherit a curated kubectl toolset with the safety
  scaffolding already in place — instead of granting them raw
  shell and praying.

**Weakness.**

1. **No deterministic analyzer floor.** Unlike
   [`k8sgpt`](../k8sgpt/), there is no built-in catalog of "look
   for these N broken-object patterns" — every investigation costs
   tokens and depends on the model picking the right `kubectl`
   call. Cost is unbounded on a messy cluster.
2. **Mutating actions exist.** The agent can `kubectl apply` /
   `delete` / `scale`. The confirmation prompts are real, but the
   blast radius of "yes" is your production cluster. Treat
   non-interactive (`--quiet` in CI) usage with the same care you
   would treat unattended `kubectl apply -f -`.
3. **Pre-1.0, fast-moving.** v0.0.x release cadence; flag names
   and config schema can shift between releases. Pin a version in
   any automation.

**When to choose.**

- **You want a chat-shaped Kubernetes assistant that runs the
  commands itself**, not just suggests them — `kubectl-ai`'s
  agent loop is exactly this.
- **You already use [`opencode`](../opencode/) /
  [`claude-code`](../claude-code/) / [`crush`](../crush/) and
  want to give them safe `kubectl` access** — `kubectl-ai
  --mcp-server` is the cleanest path; the alternative is hand-
  writing a kubectl MCP server.
- **You need fully local cluster Q&A** — point it at Ollama with a
  tool-use-capable model and the cluster never leaves your
  network.

**When not to choose.**

- **You want a one-shot triage list of broken objects with
  bounded token cost** → [`k8sgpt`](../k8sgpt/) (deterministic
  analyzers, LLM is optional narration).
- **Your problem is log volume, not cluster API state** →
  [`logai`](../logai/) (Drain template extraction over GBs).
- **You need an SRE-shaped agent that pulls in alerts, runbooks,
  Prometheus, Grafana, Datadog, and writes findings back to
  Slack / Jira / PagerDuty** → [`holmesgpt`](../holmesgpt/) — a
  much wider toolbelt at the cost of a heavier install.
- **You want a generic coding agent that can also `kubectl`** —
  use [`opencode`](../opencode/) / [`claude-code`](../claude-code/)
  with `kubectl-ai --mcp-server` mounted; do not use `kubectl-ai`
  as your IDE.

## 10. Compared to neighbors in the catalog

| Tool | Shape | Determinism floor | Mutates cluster? | MCP server? | Tool scope |
|------|-------|-------------------|------------------|-------------|------------|
| kubectl-ai | Agent REPL / one-shot | None (LLM picks every call) | Yes (with confirmation) | Yes | kubectl + bash + your MCP servers |
| [k8sgpt](../k8sgpt/) | One-shot analyzer sweep | Yes (Go analyzer catalog) | No | No (gRPC instead) | Built-in analyzers + `--explain` LLM |
| [holmesgpt](../holmesgpt/) | Investigation agent / operator | None (LLM-driven) | Yes (via Remediation toolset) | Consumes many MCP servers | 30+ data-source toolsets (Prometheus, Datadog, Grafana, …) |
| [opencode](../opencode/) + kubectl-ai MCP | Generic coding agent + cluster tools | None | Yes | Hosts the kubectl-ai MCP server | Whatever MCP servers you mount |

Decision shortcut:

- "I want one command that triages a cluster and shuts up":
  `k8sgpt analyze --explain`.
- "I want to chat with my cluster and have the agent run kubectl
  for me": `kubectl-ai`.
- "I want a long-lived SRE agent across cluster + alerts +
  runbooks + Slack": `holmesgpt`.
- "I want my existing coding agent to gain safe kubectl powers":
  `kubectl-ai --mcp-server` mounted into
  [`opencode`](../opencode/) / [`claude-code`](../claude-code/).
