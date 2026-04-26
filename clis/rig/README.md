# rig

> Snapshot date: 2026-04. Upstream: <https://github.com/0xPlaygrounds/rig>

`rig` is a **Rust** library/framework for building LLM applications:
agents, RAG pipelines, tool-using chatbots, and embedding workflows.
It occupies the niche [`langchain`](https://www.langchain.com) /
[`llama-index`](../llama-index/) hold in the Python world, but with
Rust's type system doing most of the work that decorators and dynamic
dispatch do over there. Provider clients, tool definitions, and
extractor schemas are all strongly-typed structs; you cannot build a
nonsensical pipeline because it will not compile.

It is in scope for this catalog because alongside the library it ships
**CLI examples and downstream binaries** (`rig-cli`-style agent
binaries are the canonical way users first interact with a `rig`
project), and it is increasingly the substrate underneath Rust-native
agent CLIs.

## 1. Install footprint

- As a library: `cargo add rig-core` — pulls in the core crate
  (~5 MB of Rust source after build). Add provider crates as needed:
  `rig-openai`, `rig-anthropic`, `rig-cohere`, `rig-gemini`,
  `rig-vertexai`, `rig-mistral`, `rig-ollama`, etc. — each is a
  separate crate so you only compile what you use.
- Vector-store integrations are also separate crates: `rig-lancedb`,
  `rig-qdrant`, `rig-mongodb`, `rig-neo4j`, `rig-sqlite`,
  `rig-postgres`, `rig-surrealdb`.
- `cargo run --example agent_with_tools` (and ~30 other examples in
  `rig-core/examples/`) gives you a runnable CLI agent in <60 s with
  no extra wiring.
- No daemon, no Python, single static binary per project after
  `cargo build --release`.

## 2. License

MIT (file: [`LICENSE`](https://github.com/0xPlaygrounds/rig/blob/main/LICENSE)).

## 3. What is in the box

- **Unified completion + embedding API** across ~15 providers; switch
  providers by changing the type of one variable.
- **Agent abstraction** with first-class tool calling, a pre-amble /
  system-prompt builder, and dynamic context (RAG documents stitched
  in at call time from any vector store implementing the `VectorStore`
  trait).
- **`Extractor`** — derive-macro-driven structured-output: define a
  Rust struct, get back an instance of that struct from any provider
  that supports tool calling. Effectively a Rust analogue of
  [`instructor`](../instructor/) / [`outlines`](../outlines/).
- **Multi-turn conversation state** with pluggable memory.
- **Streaming** (token + tool-call), **parallel tool execution**, and
  **MCP client** support for calling external MCP servers as tools.
- **Pipelines** module for composing chained prompts / parallel
  fan-outs without a graph DSL — just `.map`, `.parallel`, `.sequence`
  on plain Rust types.

## 4. One-liner usage

```toml
# Cargo.toml
[dependencies]
rig-core = "0.20"
rig-openai = "0.5"
tokio = { version = "1", features = ["full"] }
```

```rust
use rig::providers::openai;
use rig::completion::Prompt;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let agent = openai::Client::from_env()
        .agent("gpt-4o-mini")
        .preamble("You are a concise CLI assistant. Answer in <=2 lines.")
        .build();
    println!("{}", agent.prompt("what is rig?").await?);
    Ok(())
}
```

`cargo run` and you have a working agent CLI. The `rig-core/examples/`
directory has runnable equivalents for RAG, tool use, MCP, streaming,
and structured extraction.

## 5. Source-of-truth pin

- Verified default branch: `main`
- Latest release tag at snapshot: **`rig-vertexai-v0.3.4`** (the repo
  is a workspace; each crate releases on its own cadence — `rig-core`
  was at `v0.20.x` at snapshot)
- HEAD SHA at snapshot: `6ea80be79e61d724dcb6ec49cd56c366e22927a6`
  (committed `2026-04-24T02:59:40Z`)
- License file path: `LICENSE` (SPDX: MIT)
- Verified UTC: `2026-04-26T19:16:33Z` via `gh api repos/0xPlaygrounds/rig`

## 6. Where it fits in the zoo

`rig` is the Rust counterpart to a stack of Python entries:
[`llama-index`](../llama-index/) and [`langchain`-likes for the
pipeline / agent shape, [`instructor`](../instructor/) for structured
output, and [`pydantic-ai`](../pydantic-ai/) for the type-driven
ergonomics. Pick `rig` when (a) you want a single static binary you
can ship to a server or distribute via `cargo install`, or (b) the
rest of your stack is already Rust ([`ax`](../ax/),
[`crush`](../crush/), [`goose`](../goose/), [`codex`](../codex/)) and
you do not want a Python sidecar in the deployment story.
