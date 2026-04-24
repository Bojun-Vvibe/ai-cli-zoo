# k8sgpt

> Snapshot date: 2026-04. Upstream: <https://github.com/k8sgpt-ai/k8sgpt>
> Binary name: `k8sgpt`

`k8sgpt` is the **Kubernetes diagnostics CLI** of the catalog. It is the
first entry whose subject is a *running cluster* rather than a source
tree, a chat thread, a log file, or a packed prompt. Where
[`logai`](../logai/) mines log lines for anomalies, `k8sgpt` walks the
**live Kubernetes API** — pods, deployments, services, ingresses,
PVCs, network policies, mutating/validating webhooks, HPAs, cronjobs,
PDBs, gateways — runs a built-in catalog of analyzer rules over what
it finds, and emits a list of broken objects with a one-line
explanation per issue. The LLM stage is **optional and post-hoc**:
the analyzers find the breakage on their own, then `--explain` asks
a model to translate the structured finding into a paragraph an
on-call human can read at 3am.

That separation — deterministic analyzers in front, LLM as a
narration layer behind — is what makes `k8sgpt` distinct from a
general agent like [`OpenHands`](../openhands/) or
[`claude-code`](../claude-code/) pointed at `kubectl`. Those would
*explore* the cluster turn-by-turn and consume tokens proportional
to how messy the cluster is. `k8sgpt analyze` is one round-trip to
the API server, zero LLM tokens by default, and a stable diff-able
output shape.

The intended loop:

1. `k8sgpt analyze` — list every broken object, one line each.
2. `k8sgpt analyze --explain` — same list plus a model-written
   paragraph per issue.
3. `k8sgpt analyze --explain --with-doc` — same plus a link into the
   relevant Kubernetes docs page.
4. `k8sgpt serve` — long-running mode that exposes the same surface
   over gRPC for a controller or dashboard to poll.

## 1. Install footprint

- Single Go binary, ~50 MB, no runtime deps.
- Install: `brew install k8sgpt`, or `go install
  github.com/k8sgpt-ai/k8sgpt@latest`, or grab a release tarball
  from GitHub. Container image at `ghcr.io/k8sgpt-ai/k8sgpt`.
- Reads your existing `~/.kube/config` — the same context `kubectl`
  uses. No separate auth.
- LLM backend is configured once with `k8sgpt auth add --backend
  openai` (or `localai`, `ollama`, `azureopenai`, `cohere`,
  `amazonbedrock`, `googlevertexai`, `huggingface`, `noopai`); the
  key is stored in `~/.config/k8sgpt/k8sgpt.yaml`. Without auth,
  every analyzer feature still works — only `--explain` is gated.
- Also ships a Kubernetes Operator (`k8sgpt-operator`) that runs
  `analyze` on a schedule and writes findings as `Result` CRDs.
  The CLI and the operator share the same analyzer catalog.

## 2. License

Apache-2.0.

## 3. Models supported

LLM is **optional**, used only by `--explain`. Supported backends as
of v0.4.x: OpenAI, Azure OpenAI, Cohere, Amazon Bedrock, Google
Vertex AI, Hugging Face Inference, LocalAI (any OpenAI-compatible
endpoint, so vLLM / LiteLLM / LM Studio all work), Ollama, and a
`noopai` backend for testing the analyzer surface without a key.

The deterministic analyzer catalog covers — among others —
`Pod`, `Deployment`, `ReplicaSet`, `StatefulSet`, `Service`,
`Ingress`, `PersistentVolumeClaim`, `HorizontalPodAutoscaler`,
`PodDisruptionBudget`, `NetworkPolicy`, `MutatingWebhookConfiguration`,
`ValidatingWebhookConfiguration`, `CronJob`, `Node`,
`GatewayClass`, `Gateway`, `HTTPRoute`. Each analyzer is a Go
package under `pkg/analyzer/` — readable in an afternoon, and
straightforward to extend. Custom analyzers can also be loaded as
external gRPC plugins, which is the production extension path.

## 4. MCP support

None as of v0.4. The extension surface is **gRPC**: `k8sgpt serve`
exposes `Analyze` / `Configure` / `AddConfig` over gRPC, and
external analyzers register via the `CustomAnalyzer` plugin API.
For an MCP-shaped wrapper, run `k8sgpt serve` and write a small
adapter — there is no first-party MCP server in the repo.

## 5. Sub-agent model

None. `k8sgpt analyze` runs the entire analyzer catalog in one
sweep, parallelised across object kinds, and emits a flat list.
There is no planning loop, no tool-use turn, no agent recursion.
The only thing that recurses is the Operator's reconciliation loop,
and that is a Kubernetes-controller pattern, not an LLM agent.

## 6. Telemetry stance

**Off by default.** No analytics ping. Network egress is exactly
two endpoints: the Kubernetes API server (read-only by default) and
the configured LLM provider, and only when `--explain` is passed.
Air-gapped clusters can run `k8sgpt analyze` with no LLM backend
configured and still get the full structured findings list.

## 7. Prompt-cache strategy

For the `--explain` stage, prompts are short (one finding's
structured payload, plus a system prompt) and re-issued per finding.
The cache `--cache` flag persists explanations keyed by the analyzer
output hash, so the same broken object hit repeatedly does not
re-spend tokens. Cache backends: in-memory (default), filesystem,
S3, Azure Blob, GCS. This is the cache layer to enable when the
Operator is running on a 30-second loop in a noisy cluster.

## 8. Hot keybinds

No TUI and no REPL — `k8sgpt` is a one-shot CLI plus a long-running
gRPC `serve`. The ergonomic surface is the flag set. Most useful
shapes:

- `k8sgpt analyze` — one-line-per-issue list, no LLM.
- `k8sgpt analyze --explain` — same plus paragraph explanations.
- `k8sgpt analyze --filter Pod,Service` — restrict analyzers.
- `k8sgpt analyze --namespace prod` — restrict to one namespace.
- `k8sgpt analyze --output json` — machine-readable output for
  piping into [`jq`](../chatblade/) / dashboards / alerting.
- `k8sgpt analyze --no-cache --explain --with-doc` — fresh
  explanations with documentation links, useful for incident
  postmortems.
- `k8sgpt filters list` / `add` / `remove` — manage which analyzer
  kinds run by default.
- `k8sgpt integration activate trivy` — pull in third-party
  analyzers (Trivy for image vuln scanning is the canonical one).
- `k8sgpt serve` — gRPC server mode for the Operator or a custom
  client.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Deterministic-first triage, narration-second.
Every finding `k8sgpt` emits comes from a Go analyzer that you can
read and reason about; the LLM never *decides* what is broken, it
only describes what the analyzer already found. That property
matters in two operational contexts: (a) audits, where you need to
justify "why did the tool flag this", and (b) cost ceilings, where
you cannot afford an exploratory agent burning tokens proportional
to cluster mess. Compare with running [`claude-code`](../claude-code/)
or [`OpenHands`](../openhands/) against `kubectl` — those will work,
but the token bill is unbounded and the findings are not
reproducible across runs.

**Weakness.** Three:

1. **Read-only diagnosis, not remediation.** `k8sgpt` will tell you
   "Service `foo` has no endpoints because no pod matches selector
   `app=foo`", but it will not patch the deployment. Pair it with
   an actual agent (or a human) for the fix step.
2. **Findings quality is bounded by the analyzer catalog.** If the
   broken object is a CRD that no analyzer covers, `k8sgpt` will
   not find it. The Trivy / Prometheus / Keda integrations help,
   but custom CRDs need a custom analyzer.
3. **`--explain` quality scales with model quality.** Local Ollama
   backends often produce vague paragraphs ("the pod is failing
   because of an issue with its configuration"); production
   incident-response usage tends to land on GPT-4-class or
   Claude-Sonnet-class models, which puts the cost back on the
   table for noisy clusters unless `--cache` is on.

**When to choose.**

- **Cluster is misbehaving and you need a triage list right now.**
  `k8sgpt analyze` is faster than scrolling `kubectl get events
  --all-namespaces` and grouping by hand.
- **You want a Kubernetes-native, in-cluster watcher** that emits
  findings as CRDs for a dashboard or alerting rule to consume.
  Install the Operator; `k8sgpt analyze --output json` is the same
  data as the `Result` CR's `.spec`.
- **Your platform team needs a tool for SREs that does not require
  prompt-engineering skill.** The deterministic analyzers do the
  hard part; SREs read English paragraphs out of `--explain`.
- **You want LLM-narrated findings in an air-gapped cluster.** Wire
  `k8sgpt` to a `localai` / `ollama` backend; analyzer findings are
  identical, only the explanation prose changes quality.

**When not to choose.**

- **You want the model to *fix* the cluster, not just describe what
  is broken.** Reach for an agent with a `kubectl` tool —
  [`OpenHands`](../openhands/), [`claude-code`](../claude-code/),
  [`opencode`](../opencode/), or [`goose`](../goose/) with an MCP
  server wrapping kubectl.
- **Your data source is logs, not the live cluster API.** Use
  [`logai`](../logai/) for Drain-style template extraction over GBs
  of log text; `k8sgpt` does not parse log streams.
- **You only want to ask ad-hoc Kubernetes questions in chat
  ("why is my pod CrashLoopBackOff"), not a structured analyzer
  sweep.** Pipe `kubectl describe pod ... | mods` or use any
  catalog REPL — `k8sgpt`'s value is the analyzer catalog, and
  bypassing it for a freeform question is using the wrong tool.
- **You need cross-cluster correlation across many fleets.** That
  is a fleet-management product (Rancher, ArgoCD ApplicationSets,
  etc.), not the per-cluster diagnostic that `k8sgpt` is.
