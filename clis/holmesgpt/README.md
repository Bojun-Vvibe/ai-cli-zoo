# holmesgpt

> Snapshot date: 2026-04. Upstream: <https://github.com/robusta-dev/holmesgpt>
> Binary name: `holmes`

`holmesgpt` is the **SRE-shaped investigation agent** of the
catalog. The frame is "I have an alert / a ticket / a vague
'something is broken' — investigate and write a root-cause
report", not "I have a repo and want to edit it" (the rest of the
zoo) and not "give me the broken-object list right now"
([`k8sgpt`](../k8sgpt/)). It is a CNCF Sandbox project originally
authored by Robusta.dev.

The intended loop:

1. A signal arrives — a Prometheus AlertManager alert, a
   PagerDuty incident, an OpsGenie page, a Jira ticket, a Slack
   mention, a GitHub issue, or just `holmes ask "..."` from a
   terminal.
2. The agent picks which **toolsets** to query (Kubernetes,
   Prometheus, Grafana, Loki, Tempo, Datadog, NewRelic, Sentry,
   Elasticsearch, Postgres / MySQL / MongoDB, AWS, GCP, ArgoCD,
   Helm, Kafka, RabbitMQ, GitHub, Confluence, ServiceNow, …) and
   runs read-only queries against them.
3. It writes a structured root-cause report and (optionally)
   writes that report back to the source channel — Slack
   message, Jira comment, PagerDuty note, GitHub PR comment.
4. In **operator mode**, the same agent runs continuously inside
   Kubernetes, executes scheduled health checks against any
   data source it is wired to, and proactively notifies on
   regressions.

## 1. Install footprint

- Two distinct shapes:
  - **CLI** (`holmes`): `pipx install holmesgpt`, or pull the
    Docker image `robustadev/holmes-cli`. Used for ad-hoc
    `holmes ask "..."` and for one-shot pulls from PagerDuty /
    OpsGenie / Jira / GitHub.
  - **In-cluster Operator**: Helm-installed (`helm repo add
    holmes https://robusta.dev/charts && helm install holmes
    holmes/holmes`); runs as a long-lived pod, ingests
    AlertManager webhooks, optionally consumes the same toolset
    surface against any external data source.
- Python project under the hood (`holmesgpt` PyPI package); the
  Docker image bakes in a `kubectl` binary so the Kubernetes
  toolset works out of the box inside the operator pod.
- Configuration via `~/.holmes/config.yaml` (CLI) or a Helm
  values file (operator). Toolsets are opt-in — the default
  install ships with Kubernetes + Prometheus + a generic
  internet/runbooks toolset wired up; everything else
  (Grafana, Datadog, GitHub, Sentry, etc.) needs explicit
  config + credentials.

## 2. Repo, version, license

- Repo: <https://github.com/robusta-dev/holmesgpt>
- Version: **0.25.0** (2026-04).
- License: Apache-2.0. License file at the repo root:
  [`LICENSE`](https://github.com/robusta-dev/holmesgpt/blob/master/LICENSE).
- Language: Python.
- Governance: CNCF Sandbox project as of 2025.

## 3. Models supported

Provider-agnostic via LiteLLM under the hood. Supported as of
0.25.x: OpenAI, Anthropic, Azure OpenAI, AWS Bedrock, Google
Vertex AI / Gemini, and any OpenAI-compatible endpoint (Ollama,
vLLM, LiteLLM proxy, OpenRouter). Function-calling is required —
the agent loop is built on tool-use; chat-only models are not a
supported configuration.

The recommended production model class is GPT-4-class or
Sonnet-class — toolset breadth means the model has to pick from a
large action surface, and smaller models tend to mis-pick.

## 4. MCP support

**Yes — heavily as a client.** Many of the built-in toolsets are
themselves MCP servers (AWS, Azure, Confluence, GitHub, Sentry,
Splunk, Jenkins, Prefect, MariaDB, "Kubernetes Remediation", and
others). The `holmes` agent multiplexes across them, presenting
a flattened tool surface to the LLM.

There is no first-party "expose holmes as an MCP server" mode as
of 0.25 — its consumption surface is its own CLI, REST API, and
operator webhooks, not an MCP endpoint another agent would mount.

## 5. Sub-agent model

Single agent loop per investigation. What looks like sub-agency
in practice is **toolset composition**: each toolset
(Kubernetes, Prometheus, Datadog, …) is a self-contained set of
tools the agent can call. Operator mode adds a separate
long-running scheduler that *spawns* investigations on a
schedule or in response to alerts; each investigation is still a
single-agent loop.

## 6. Telemetry stance

**Off in the OSS codebase.** No analytics ping from the open-
source `holmes` binary or operator. Network egress is exactly
the configured LLM provider, the toolsets you have wired up
(Kubernetes API, Prometheus, Datadog HTTP, etc.), and any
opt-in write-back integrations (Slack webhook, Jira REST,
PagerDuty notes). The hosted commercial offering from Robusta
adds its own telemetry; the OSS install does not.

## 7. Token / context strategy

The headline design choice is **petabyte-aware tool output
handling**:

- Per-tool memory limits cap how much a single tool call can
  return.
- Large tool outputs stream to disk and are referenced by ID
  rather than inlined into context.
- "Output transformers" do server-side filtering and JSON tree
  traversal so the agent gets the relevant slice instead of the
  raw 50MB log dump.
- An automatic output budgeter trims responses before they hit
  the model's context window.

This is the property that makes `holmesgpt` operationally
distinct: the same investigation against a chatty cluster will
make a generic coding agent OOM the model context; `holmes`
holds the line.

## 8. Hot keybinds

CLI is line-oriented (no full-screen TUI). Useful shapes:

- `holmes ask "why is the checkout pod crashing?"` — one-shot
  investigation against the configured toolsets, prints the
  report.
- `holmes investigate prometheus-alertmanager <alert-name>` —
  pull a specific alert from AlertManager and investigate it.
- `holmes investigate jira <ticket-id>` — same for a Jira
  ticket; can be configured to write the report back as a Jira
  comment.
- `holmes investigate pagerduty <incident-id>` /
  `holmes investigate opsgenie <alert-id>` /
  `holmes investigate github <repo>/<issue>`.
- `holmes toolset list` / `holmes toolset enable <name>` /
  `holmes toolset config <name>` — manage which toolsets are
  active.
- `holmes serve` — run as a REST server (the operator pod uses
  this internally).

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Toolset breadth + petabyte-aware output
handling, in a single agent that lives where the alert lives.**
No other catalog entry covers Prometheus, Grafana, Loki, Tempo,
Datadog, NewRelic, Sentry, Elasticsearch, Postgres, MySQL,
MongoDB, Kafka, RabbitMQ, AWS, GCP, ArgoCD, Helm, Confluence,
ServiceNow, GitHub, Jira, PagerDuty, OpsGenie, *and* Kubernetes
under one tool surface, with the engineering work to make
multi-GB tool outputs not blow the model context. The closest
neighbors ([`k8sgpt`](../k8sgpt/), [`kubectl-ai`](../kubectl-ai/))
are Kubernetes-only.

**Weakness.**

1. **Heavy install for a CLI.** The CLI alone is fine, but the
   product really wants the operator + Helm + a Kubernetes
   cluster + at least one observability stack wired up. If
   you only have a laptop and a `kubectl context`, the value
   over `kubectl-ai` is small.
2. **Cost shape is unbounded by design.** A real investigation
   can fan out across N toolsets and burn meaningful tokens
   per incident. The output budgeter caps context but not call
   count; budget alarms on the LLM provider side are
   non-optional in production.
3. **Investigation quality scales with toolset coverage.** A
   `holmes` install with only the Kubernetes toolset wired up
   will reliably under-perform `k8sgpt --explain` on the same
   cluster, because all of `holmes`'s value is multi-source
   correlation. Plan to wire ≥3 toolsets or pick something
   simpler.

**When to choose.**

- **You run an SRE / on-call practice and want one agent that
  ingests alerts from multiple sources and writes investigations
  back to where the human looks.** No other catalog entry has
  the alert-ingest + write-back surface area.
- **Your incidents routinely cross Kubernetes + observability +
  ticketing.** `holmesgpt` is shaped for that exact pattern.
- **You want a CNCF-governed, vendor-neutral SRE agent**, not a
  cloud-vendor-specific one.

**When not to choose.**

- **You only need to look at one cluster, occasionally.** The
  install tax is not worth it; use [`k8sgpt`](../k8sgpt/) for
  bounded triage or [`kubectl-ai`](../kubectl-ai/) for chat-
  shaped Kubernetes Q&A.
- **You want a coding agent that edits source files.** Wrong
  shape entirely — see [`opencode`](../opencode/),
  [`aider`](../aider/), [`claude-code`](../claude-code/).
- **You need a *grader* for prompt regressions** →
  [`promptfoo`](../promptfoo/).
- **Your data source is raw log volume, not structured
  observability** → [`logai`](../logai/) (Drain templating +
  anomaly detection over GBs of log text); pair with `holmes`
  if you want the narrative summary on top.

## 10. Compared to neighbors in the catalog

| Tool | Scope | Toolset breadth | Output-budgeting | Operator/scheduler mode | Write-back integrations |
|------|-------|-----------------|------------------|-------------------------|-------------------------|
| holmesgpt | Cross-stack SRE investigation | 30+ toolsets | Yes (per-tool limits, streaming, transformers) | Yes (in-cluster operator + scheduled health checks) | Slack, Jira, PagerDuty, OpsGenie, GitHub |
| [k8sgpt](../k8sgpt/) | Kubernetes-only triage | Kubernetes analyzers + Trivy/Keda integrations | Per-finding (analyzers emit small structured payloads) | Yes (k8sgpt-operator with `Result` CRDs) | None first-party |
| [kubectl-ai](../kubectl-ai/) | Kubernetes chat agent | kubectl + bash + user-mounted MCP servers | No (relies on LLM context window) | No | None |
| [opencode](../opencode/) + MCP servers | Generic coding agent | Whatever MCP servers you mount | No | No | Through MCP servers only |

Decision shortcut:

- "Triage a misbehaving cluster, bounded cost, no LLM
  required for the floor": `k8sgpt`.
- "Chat with my cluster from a terminal, agent runs kubectl
  itself": `kubectl-ai`.
- "Investigate a multi-source production incident and write
  the RCA back to Jira / Slack / PagerDuty, with an in-cluster
  operator that watches 24/7": `holmesgpt`.
- "Mount cluster tools into my generic coding agent":
  [`kubectl-ai`](../kubectl-ai/) `--mcp-server` inside
  [`opencode`](../opencode/) / [`claude-code`](../claude-code/).
