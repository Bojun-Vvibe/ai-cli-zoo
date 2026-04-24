# seagoat

> Snapshot date: 2026-04. Upstream: <https://github.com/kantord/SeaGOAT>

"**A code search engine for the AI age.**" `seagoat` is a **local-first
semantic code search** daemon plus a `gt` query CLI: it embeds your
repository with sentence-transformers, stores vectors locally
(ChromaDB), and answers natural-language queries like
`gt "where do we round numbers"` with ranked file:line hits. Crucially,
the *vector* hit list is **fused with `ripgrep` literal hits and a
regex pass**, so it degrades to plain grep when the query is exact and
to semantic search when it is fuzzy — one CLI for both shapes.

## 1. Install footprint

- `pipx install seagoat` (recommended). Requires Python 3.11+ and
  `ripgrep` on `PATH`; `bat` is optional but auto-used for highlighted
  output.
- Two commands: `seagoat-server start /path/to/repo` boots a
  long-lived background indexer + HTTP server; `gt "<query>"` (alias
  `seagoat`) issues queries against it.
- Index lives under `~/.local/share/seagoat/` (XDG-respecting); first
  full index of a large repo takes minutes and CPU, then incremental.
- Pure-local: ships its own embedding model (`sentence-transformers`),
  no API key required for the default config.

## 2. Repo + version + license

- Repo: <https://github.com/kantord/SeaGOAT>
- Latest release: **v0.54.17**
- License: **MIT** —
  <https://github.com/kantord/SeaGOAT/blob/main/LICENSE>
- Default branch: `main`

## 3. Models supported

Local only by default — sentence-transformers embedding model bundled,
ChromaDB vector store on disk. No cloud LLM call in the default flow;
the query path is pure vector + lexical search, not "ask GPT about
your code". You can swap the embedding backend via config, but there
is no first-class "summarize the result with an LLM" mode (compose
with [`llm`](../llm/) / [`mods`](../mods/) / [`aichat`](../aichat/)
yourself if you want that).

## 4. MCP support

None. SeaGOAT exposes an HTTP API (used by the `gt` CLI and the
shipped editor integrations for Vim / VS Code / Emacs), not MCP. If
you want it inside an MCP-aware agent, wrap the HTTP API in a small
MCP server yourself.

## 5. Sub-agent model

None. SeaGOAT is **not an agent** — it is a search index with a
query CLI. Its job ends at "here are the top-N relevant
file:line:snippet hits." Pair it with a coding agent
([`aider`](../aider/), [`opencode`](../opencode/),
[`claude-code`](../claude-code/)) that consumes those hits as
context.

## 6. Telemetry stance

Off. No analytics. The daemon binds locally; embedding happens
locally; the query path is a local HTTP call. The only network egress
is whatever editor integration or pipeline you wire on top.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Semantic + literal + regex fusion in one
ranking.** A query like `gt "function that retries with exponential
backoff"` returns vector-similar code; `gt "RETRY_BACKOFF_MS"`
returns the exact-string hits ripgrep would have found; `gt
"retry.*backoff"` runs as regex. You do not have to pre-decide
"should I grep or should I semantic-search?" — SeaGOAT runs both and
fuses the rankings. That makes it a genuinely Unix-pipeline-friendly
replacement for grep on large repos where you do not always remember
the exact identifier.

**Weakness.** The local daemon model means one running
`seagoat-server` per repo you actively query — easy to forget and let
rot. First-time indexing on a >1M-line repo is slow (CPU + RAM
heavy); the bundled embedding model is good but not best-in-class for
code (no CodeBERT / Voyage code-3 by default). No first-party MCP
server, so wiring it into [`opencode`](../opencode/) /
[`claude-code`](../claude-code/) takes a small adapter. README notes
macOS support is partly tested, Windows support needs help — Linux
is the supported tier.

**When to choose.** You work in a large repo, you find yourself
guessing at function names to grep for, and you want a *local*
semantic index (no source code uploaded to a vendor) that still
behaves like a Unix tool when the query is exact. Pick this over
`repomix` / `files-to-prompt` when your problem is *finding* the
right files, not packing them once you know which they are. Pair
SeaGOAT (for retrieval) with `aider` or `claude-code` (for editing)
to keep the context window small. Skip it if your codebase is small
enough that `rg` is already fast enough, or if you need
cloud-embedded code search across many repos at once.
