# e2b

> Snapshot date: 2026-04. Upstream: <https://github.com/e2b-dev/E2B>

"**Open-source, secure environment with real-world tools for
enterprise-grade agents.**" E2B is a **remote sandbox-as-a-service**:
the `e2b` CLI manages **Firecracker-microVM-backed cloud sandboxes**
that an LLM agent can spawn programmatically to run arbitrary code,
shell commands, or browser sessions in milliseconds. Where
[`container-use`](../container-use/) gives you parallel sandboxes on
your own host and [`codel`](../codel/) gives you a single
docker-bundled agent runtime, E2B is the **hosted sandbox plane** the
big agent frameworks (`smolagents`, `crewai`, langgraph, custom code)
target when they need real, isolated, network-attached compute per
task.

## 1. Install footprint

- `npm install -g @e2b/cli` (the `e2b` command). The shipping
  language SDKs (`npm i e2b`, `pip install e2b`) embed the same
  control-plane client.
- Single static Node binary surface: `e2b auth login`, `e2b sandbox
  list / kill / connect`, `e2b template build / push / list`, `e2b
  template init` (scaffolds a Dockerfile that becomes a sandbox
  template).
- Requires an `E2B_API_KEY` (free tier on signup); CLI calls the
  hosted control plane to create / destroy sandboxes.
- Self-hosting path exists (the infra repo is `e2b-dev/infra`,
  Apache-2.0) for organisations that need on-prem; the OSS CLI works
  identically against either.

## 2. Repo + version + license

- Repo: <https://github.com/e2b-dev/E2B>
- Latest release: **e2b@2.19.1** (CLI tag in the monorepo)
- License: **Apache-2.0** —
  <https://github.com/e2b-dev/E2B/blob/main/LICENSE>
- Default branch: `main`
- CLI source lives under `packages/cli/` in the monorepo.

## 3. Models supported

None directly. E2B is the **execution substrate**, not a model client.
You bring the agent (`smolagents`, `crewai`, `langgraph`, your own
code, or a coding CLI in this catalog that supports custom sandboxes)
and E2B provides the place for that agent's generated code / shell
commands / browser actions to actually run.

## 4. MCP support

Not in the CLI itself. The control surface is the SDK / CLI plus a
REST API; community MCP wrappers exist for exposing E2B sandboxes as
MCP tools to agents like [`opencode`](../opencode/) /
[`claude-code`](../claude-code/), but they are not first-party as of
this snapshot.

## 5. Sub-agent model

None. E2B does not orchestrate agents — agents orchestrate E2B. A
typical pattern: a `crewai` crew spawns one E2B sandbox per worker
agent, each agent does its work in its own VM, results stream back,
sandboxes are killed at the end of the task.

## 6. Telemetry stance

On (by definition — the CLI is a control plane for a hosted service).
Sandbox lifecycle, resource usage, and your API key are visible to
the E2B control plane. Egress *from inside* a sandbox is whatever the
sandbox itself does (the LLM-generated code can `curl` anything by
default; lock down with template-level egress rules). For air-gapped
use, run the self-hosted infra (`e2b-dev/infra`) and point the CLI
at it.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Sub-second Firecracker microVM spawn for
LLM-generated code.** Where Docker-based sandboxes ([`codel`](../codel/),
[`OpenHands`](../openhands/), [`container-use`](../container-use/))
take seconds to start a fresh container, E2B sandboxes start in
~150 ms because they are pre-warmed Firecracker VMs. That makes
"spawn a fresh sandbox per tool call" feasible — every Python
`exec()` from your agent gets its own VM, no state-leak between
calls, no hand-rolled cleanup. Pair that with custom templates
(your own Dockerfile + E2B's runtime hooks) and you get a per-task
reproducible env without any of the local Docker socket / root
escalation footguns that bite docker-mount agents.

**Weakness.** Hosted-by-default — your sandbox lives on E2B's
infrastructure, your code runs there, you pay per second of
sandbox uptime. The free tier is generous but real workloads cost
real money. Self-hosting is supported but the install is non-trivial
(it's a small Kubernetes-shaped platform). The CLI alone does not do
anything useful — you need to write SDK code (Python or TS) to drive
the sandboxes, the CLI is only for auth / list / template management
/ debug. There is no built-in "and here is an agent" — that is
deliberate, but it means E2B is *not* a drop-in for `claude-code` /
`opencode` / `codex` if you wanted a one-binary coding session.

**When to choose.** You are *building* an agent (in `smolagents` /
`crewai` / `langgraph` / your own framework) that needs to execute
LLM-generated code safely, in parallel, with hard isolation, and you
do not want to operate a container fleet yourself. Pick this over
[`container-use`](../container-use/) when you need cloud-scale,
sub-second cold starts, and you are happy with hosted infra; pick
[`container-use`](../container-use/) instead when you want everything
on your laptop with `git`-branch-per-agent semantics. Pick this over
[`codel`](../codel/) when you need *many* concurrent sandboxes rather
than one bundled all-in-one runtime. Skip it if you only want a
single coding-CLI on your local machine — none of E2B's value
materialises until you have an agent that programmatically spawns
sandboxes.
