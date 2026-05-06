# weaver

> **weaver** — open-telemetry/weaver, the OpenTelemetry project's
> official CLI for *semantic conventions*: define your telemetry
> schema (metric names, attribute keys, event shapes) once in YAML,
> then generate validated docs, code, schemas, and policy checks for
> every language and pipeline stage. Pinned to **v0.23.0**,
> Apache-2.0 — license file:
> [LICENSE](https://github.com/open-telemetry/weaver/blob/main/LICENSE).

Source: <https://github.com/open-telemetry/weaver>

## TL;DR

OpenTelemetry has a hard problem the auto-instrumentation libraries
hide: **what should this attribute be called?** `http.method` or
`http.request.method`? `db.system` or `db.system.name`? Get it wrong
and your alerts, dashboards, and exemplar-driven traces stop
correlating. The OTel project answers this with a *semantic
conventions* registry — versioned YAML files describing every
canonical attribute, metric, event, and span the spec blesses.

`weaver` is the Rust binary that operates on those YAML files:

- **`weaver registry check`** — lint a registry: every attribute
  has a type, every reference resolves, naming follows the SemConv
  rules, deprecated entries declare a successor.
- **`weaver registry generate`** — render a registry through
  Jinja2-based templates into Markdown docs, Go / Rust / Python
  constants, JSON schemas for log validators, OTLP-compatible YAML
  for the Collector's `transform` processor, etc. The template set
  for upstream OTel docs lives in `semantic-conventions/templates/`
  and is itself driven by weaver.
- **`weaver registry resolve`** — flatten the multi-file registry
  into one canonical JSON document downstream tools can consume
  without re-implementing the include/extend resolver.
- **`weaver registry diff`** — diff two registry versions and
  report breaking changes (renamed attribute, type change, removed
  metric) with a stability classification — the input to a release
  note or a "should this be a major bump" decision.
- **`weaver registry stats`** — coverage report: how many groups,
  how many attributes, how many `stable` vs `experimental` vs
  `deprecated`.
- **`weaver registry live-check`** — point at a stream of OTLP
  traffic and report which attributes in the wire data are *not* in
  the registry (drift detection between code and the canonical
  schema).

The reason to care: SemConv used to be a hand-edited Markdown
registry that drifted from the constants in language SDKs. Weaver
makes the YAML the source of truth, generates everything else from
it, and lets *application teams* maintain their own internal
registry on top of the upstream one (your company's `tenant.id`,
`feature.flag.key`, `mfa.method` attributes alongside the OTel ones)
with the same lint + codegen + diff workflow.

## Install

```bash
# Cargo (the canonical install path)
cargo install weaver_forge --locked

# Pre-built binaries (Linux x64 / arm64, macOS x64 / arm64, Windows)
# https://github.com/open-telemetry/weaver/releases/tag/v0.23.0

# Docker image
docker run --rm -v "$(pwd):/home/weaver/source" \
  otel/weaver:0.23.0 registry check -r /home/weaver/source

# Build from source
git clone https://github.com/open-telemetry/weaver
cargo install --path crates/weaver
```

## Example commands

```bash
# Lint a SemConv registry (your local copy of the OTel registry, or
# your company's internal one)
weaver registry check -r ./model

# Generate Markdown docs for the registry into ./docs
weaver registry generate \
  --registry ./model \
  --templates ./templates \
  markdown ./docs

# Generate Go constants
weaver registry generate \
  --registry ./model \
  --templates ./templates/go \
  go ./pkg/semconv

# Diff two registry versions for breaking-change detection
weaver registry diff \
  --baseline-registry https://github.com/open-telemetry/semantic-conventions/archive/refs/tags/v1.27.0.zip[model] \
  --registry ./model

# Resolve includes and emit one flat JSON document
weaver registry resolve -r ./model --output resolved.json
```

## Niche it occupies

**OpenTelemetry semantic-conventions toolchain** — different niche
from the rest of the catalog. Closest neighbours:

- [`otel-collector`](../otel-collector/) — the OTLP pipeline daemon
  that *moves* telemetry. Orthogonal: the collector ingests / routes
  / exports OTLP; weaver defines the *vocabulary* the OTLP payloads
  should use. Compose: a `transform` processor in the collector
  consumes weaver-generated YAML to rename non-conformant attributes
  on the wire.
- [`otel-cli`](../otel-cli/) /
  [`otel-desktop-viewer`](../otel-desktop-viewer/) — debug clients
  that *send* or *display* OTLP. Orthogonal: those produce / inspect
  OTLP traffic, weaver describes what *should* be in it.
- [`buf`](../buf/) — schema toolchain for protobuf (lint, breaking-
  change detection, codegen, registry hosting). Closest analogy in
  shape; weaver is the same idea but for OTel SemConv YAML instead
  of `.proto` files. The lint / diff / generate / publish verb set is
  deliberately similar.
- [`ast-grep`](../ast-grep/) / [`semgrep`](../semgrep/) /
  [`comby`](../comby/) — code search / refactor. Compose with
  weaver: when `weaver registry diff` reports a rename, an
  `ast-grep` rule rewrites every `tracer.start_span("http.method")`
  to `"http.request.method"` across the codebase.
- [`vacuum`](../vacuum/) / [`vale`](../vale/) — linters for OpenAPI
  and prose. Same shape (declarative rules over a structured
  document), different domain.

Pairs cleanly with [`opentelemetry`](../otel-collector/) language
SDKs (the generated constants drop into the SDK call sites), with
[`vector`](../vector/) / [`fluent-bit`](../fluent-bit/) (sister
pipelines that can consume weaver-generated mapping tables to
normalise legacy logs into SemConv-compliant attributes), and with
[`logfire`](../logfire/) / [`langfuse`](../langfuse/) /
[`weave`](../weave/) (LLM-trace tools that benefit from a
weaver-defined GenAI SemConv registry — the upstream
`gen_ai.request.model`, `gen_ai.response.input_tokens`, etc.
attributes are themselves weaver-generated).

Caveats: weaver is *only useful if you treat your telemetry vocabulary
as a versioned schema* — a team that names attributes ad-hoc per
service will find the lint friction higher than the dashboard payoff;
template authoring is non-trivial (Jinja2 + a weaver-specific
filter library, debugging via `--debug` and reading the generated
artefact); the registry-resolve format is still pre-1.0 (post-0.23
expect breaking changes — pin both `weaver` and the SemConv version
together), and the upstream OTel SemConv registry itself is large
(thousands of attributes) so a first run of `weaver registry check`
on it takes a few seconds.

## Citation

- Repo: <https://github.com/open-telemetry/weaver>
- Latest release: **v0.23.0**
- License: **Apache-2.0**
- License file: [LICENSE](https://github.com/open-telemetry/weaver/blob/main/LICENSE)
