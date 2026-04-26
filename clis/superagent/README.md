# superagent

> Snapshot date: 2026-04. Upstream: <https://github.com/superagent-ai/superagent>

A drop-in **AI runtime firewall** that sits in front of your LLM endpoint and
blocks prompt injections, data leaks, jailbreaks, off-topic abuse, and policy
violations before the request reaches the model — and again on the way back
before the response reaches the user. Originally an open-source agent
framework; pivoted in 2025 to a focused security-and-compliance gateway.

The ergonomic move is one line: change your `OpenAI(base_url=...)` to point at
the local Superagent proxy instead of `api.openai.com`. The proxy speaks
OpenAI's chat-completions wire format, runs a pipeline of detectors against
the prompt, forwards to the real provider if it passes, and runs another
pipeline against the response on the way back. Your application code does
not change.

## 1. Install footprint

- **Node SDK / proxy:** `npm install superagent-ai` (the canonical entry
  point as of `node-v0.0.9`, 2025-09).
- **Container:** `docker run -p 8080:8080 superagentai/superagent` boots the
  proxy with the default policy; mount a YAML to override.
- **CLI:** `npx superagent init` scaffolds a `superagent.yaml` policy file
  in the current directory.
- ~50 MB Node install, no Python deps. The detectors run in-process by
  default (no third-party guardrail-as-a-service in the data path); a
  hosted control plane is available but explicitly optional.

## 2. Repo, version, license

- Repo: <https://github.com/superagent-ai/superagent>
- Version checked: **`node-v0.0.9`** (released 2025-09-14; `main` at
  `5adc62d`, 2026-04-11).
- License: **MIT**. License file at the repo root: `LICENSE`.
- npm package: `superagent-ai` (do not confuse with the unrelated
  `superagent` HTTP client).

## 3. What it actually does

Three deployment shapes, same detector pipeline:

1. **Sidecar proxy.** `docker run superagentai/superagent` exposes
   `http://localhost:8080/v1/chat/completions`. Point your existing
   `OpenAI` / `Anthropic` / `Bedrock` client at it. Every request is
   inspected, every response is inspected, every block is logged.
2. **In-process middleware.** `import { guard } from 'superagent-ai'`,
   wrap your `model.generate(...)` call. No proxy hop; same policy YAML.
3. **MCP tool.** Expose the guard as an MCP server so MCP-aware agent
   CLIs ([`opencode`](../opencode/), [`claude-code`](../claude-code/),
   [`crush`](../crush/)) can call `guard.check(prompt)` as an explicit
   tool before they invoke a model.

The policy YAML lists detectors and their actions:

```yaml
detectors:
  - name: prompt_injection
    action: block          # block | redact | flag
  - name: pii
    action: redact
    fields: [email, ssn, credit_card]
  - name: ban_topics
    topics: [violence, self_harm, competitors]
    action: block
  - name: secrets
    action: block          # API keys, AWS creds, private keys
  - name: ban_substrings
    substrings: ["INTERNAL ONLY"]
    action: block
output_detectors:
  - name: hallucination
    action: flag
  - name: factual_consistency
    sources_field: context
    action: flag
```

Blocked requests return a 4xx with a structured reason; flagged requests
pass through but emit a webhook / OTel span.

## 4. MCP support

Yes — both directions. The guard exposes an MCP server (`superagent mcp
serve`) so agent CLIs can call it as a tool, and the proxy itself can
inspect MCP traffic when an MCP server is run behind it.

## 5. Sub-agent model

None. Superagent is *not* an agent framework anymore — it is a guard.
Pair with a real agent runner ([`crewai`](../crewai/),
[`pydantic-ai`](../pydantic-ai/), [`openai-agents-python`](../openai-agents-python/),
[`agentscope`](../agentscope/)) and put Superagent in front.

## 6. Telemetry stance

Off in self-hosted mode. The OSS proxy emits OTel spans only to the
exporter you configure (`OTEL_EXPORTER_OTLP_ENDPOINT`), no analytics back
to the vendor. The hosted control plane at `superagent.sh` is a separate
opt-in product; the OSS image works without ever calling it.

Detector models run locally where possible (small classifiers for prompt
injection, PII via regex + Presidio-style NER, secrets via entropy +
pattern). LLM-as-judge detectors (hallucination, factual consistency)
call out to whatever judge model the policy declares — that judge sees
the prompt and the candidate response, so for sensitive workloads point
the judge at a local model.

## 7. Token / context strategy

Detector pipelines run sequentially by default; expensive judge calls
are gated behind cheap classifiers. Latency overhead is 5–80 ms per
request on CPU for the regex / classifier detectors; LLM-as-judge
detectors add a full extra round-trip and should be enabled
selectively. The proxy supports streaming — detectors run on the
prompt before the upstream call begins, and on the response after
the stream completes (so streaming clients see no extra TTFT).

## 8. Hot keybinds

N/A — Superagent is a service, not a TUI. The CLI surface is
`superagent init`, `superagent test policy.yaml`, and `superagent
proxy --policy policy.yaml`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Wire-compatible OpenAI-shape proxy** — you keep
your existing client code unchanged, change one base URL, and get an
auditable guard layer with redaction, blocking, and span-level logs.
The MIT licence and "no third-party service in the data path" defaults
make it deployable inside regulated environments where
[`llm-guard`](../llm-guard/)-style in-process libraries are too
intrusive and managed Bedrock Guardrails / NeMo Guardrails are
ruled out by data-egress policy.

**Weakness.** The 2025 pivot from agent framework to guard means
older blog posts, tutorials, and starred examples (the repo has 6.5k
stars largely from the framework era) describe a product that no
longer exists. Anchor on the current README, not the search results.
Detector quality is bounded by the underlying classifiers — novel
multi-turn jailbreaks slip past, so pair with adversarial testing
([`deepteam`](../deepteam/)) before betting on it as a sole control.
Hallucination / factual-consistency detectors require an LLM judge
and add a full round-trip per request — budget accordingly.

**When to choose.**
- You ship an LLM-backed product to enterprise customers and need a
  **provable** guardrail layer that is self-hosted, MIT, and sits on
  the wire (not inside your application process).
- You want **OpenAI-API-compatible proxy ergonomics** — flip a base
  URL, keep all SDKs.
- You need **bidirectional inspection** (prompt and response) with a
  single policy file, not two separate libraries.

**When to skip.**
- You want guardrails **inside your Python process** with no proxy hop
  → use [`llm-guard`](../llm-guard/).
- You want **adversarial testing** of your model, not runtime defence
  → use [`deepteam`](../deepteam/) or [`garak`](../garak/).
- You want a **managed** guardrail service (no ops, vendor SLA) →
  AWS Bedrock Guardrails or NVIDIA NeMo Guardrails hosted.
- You want an **agent framework** — Superagent used to be one; it is
  not anymore. Use [`crewai`](../crewai/) /
  [`pydantic-ai`](../pydantic-ai/) /
  [`openai-agents-python`](../openai-agents-python/) instead.

## 10. Compared to neighbors in the catalog

| Tool | Shape | Wire compat | Self-hosted | Adversarial testing |
|------|-------|-------------|-------------|---------------------|
| superagent | OpenAI-shape proxy + in-process SDK | OpenAI chat-completions | Yes (MIT) | Runtime defence only |
| [llm-guard](../llm-guard/) | In-process Python library | Any (you call it) | Yes (MIT) | Runtime defence only |
| [deepteam](../deepteam/) | Adversarial harness | Any callable | Yes (Apache-2.0) | Yes (red-team) |
| [garak](../garak/) | Vulnerability scanner CLI | Any provider | Yes (Apache-2.0) | Yes (probes) |
| [portkey-gateway](../portkey-gateway/) | Multi-provider gateway w/ guardrails | OpenAI-compatible | Yes (MIT) | Limited |

Decision shortcut:

- "Wire-level proxy, OpenAI-shape clients unchanged, MIT" →
  `superagent`.
- "In-process Python library, no proxy hop" → `llm-guard`.
- "Adversarial red-team my model before shipping" → `deepteam` or
  `garak`.
- "Multi-provider routing **plus** guardrails at the same hop" →
  [`portkey-gateway`](../portkey-gateway/).
