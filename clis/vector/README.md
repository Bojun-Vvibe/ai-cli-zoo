# vector

- **Repo:** https://github.com/vectordotdev/vector
- **Version:** v0.55.0
- **License:** [LICENSE](https://github.com/vectordotdev/vector/blob/master/LICENSE) (MPL-2.0)
- **Category:** Observability pipeline / logs+metrics+traces router

## What it is

Vector is a single static Rust binary that ingests, transforms, and ships
observability data (logs, metrics, and increasingly traces) between any pair of
sources and sinks. You declare a pipeline in TOML/YAML/JSON as a DAG of
`sources` → `transforms` → `sinks`, and `vector` runs it as a long-lived
agent (host-level), aggregator (regional collector), or sidecar. Transforms
range from cheap reshaping (`remap`, `filter`, `route`) to a full embedded
expression language (VRL — Vector Remap Language) for parsing, redaction,
enrichment, and sampling without spinning up a separate stream processor.

## Why it fits an AI-native workflow

- **One binary in front of every LLM-touching service** — agent traces, prompt
  logs, token-usage metrics, and tool-call audit events all funnel through one
  pipeline before fanning out to S3 (cheap cold storage), ClickHouse (analytics),
  and a vendor SIEM, so you stop wiring three SDKs into every Python service.
- **VRL for prompt/PII redaction at the edge** — strip API keys, customer
  names, and embedded base64 blobs out of prompt logs *before* they leave the
  host, which makes "can we log full prompts?" answerable with "yes, redacted
  by the agent."
- **Adaptive sampling for expensive trace streams** — `sample` and `throttle`
  transforms cap noisy agent retries so you do not pay vendor ingest for the
  10000th identical "rate limited, backing off" span; combined with VRL
  `tail_sampling`-style logic you keep every error trace and 1% of successes.
