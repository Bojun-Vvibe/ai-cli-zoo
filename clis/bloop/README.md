# bloop

> Snapshot date: 2026-04. Upstream: <https://github.com/BloopAI/bloop>
> Binary name: `bleep` (server) + Tauri desktop UI

`bloop` is a **fast local code search engine written in Rust** with an
LLM-powered natural-language layer on top. Unlike
[`seagoat`](../seagoat/) (semantic search served straight to a coding
agent) or [`gitingest`](../gitingest/) (whole-repo dump for an LLM),
`bloop` indexes your repos with `tantivy` + `tree-sitter` symbol
extraction + a local embedding model and exposes a hybrid retrieval
surface: **regex + symbol search + semantic search**, fused into one
ranked result list. The natural-language layer ("explain how auth
works in this repo") plans a multi-step retrieval over that index and
returns answers with line-anchored citations into the source.

The shape that distinguishes it from the rest of the catalog: bloop
runs **continuously as a local daemon (`bleep`)** indexing your
checkouts, with a desktop UI as the primary surface and a documented
HTTP API underneath, so you can also wire it into a coding agent as
the search-and-citation backend instead of letting the agent grep blindly.

## 1. License

Apache-2.0. Default branch is `oss` (the open-source release line).

## 2. Latest version

`v0.6.5` (released 2024-04-23). Project is in maintenance — useful as
a self-hostable code-search substrate, not as a fast-moving agent.

## 3. Install

```bash
# macOS desktop app (recommended):
brew install --cask bloop

# Or build from source:
git clone https://github.com/BloopAI/bloop
cd bloop
cargo build --release -p bleep        # server only
# binary lands at ./target/release/bleep

# Daemon mode (no UI):
./bleep --source-dir ~/code --host 127.0.0.1 --port 7878
```

LLM keys (OpenAI / Anthropic) go in the desktop app's Settings or via
env vars when running the server headless.

## 4. When to choose

- You want a **persistent, locally-indexed, citation-returning code
  search service** across many repos that you can query from a UI
  *and* point a coding agent at via HTTP.
- You want **regex + symbol + semantic** results fused into one list
  rather than picking one tool per question.
- You need everything to stay on-box (Apache-2.0, self-hosted, local
  embeddings).
- Avoid if you want an active project — pick [`seagoat`](../seagoat/)
  for an actively-maintained semantic-search-for-agents surface, or
  [`serena`](../serena/) if you want LSP-grade symbolic edit tools
  exposed over MCP instead of search.
