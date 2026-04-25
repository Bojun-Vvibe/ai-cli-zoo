# marimo

> Snapshot date: 2026-04. Upstream: <https://github.com/marimo-team/marimo>

"**A reactive notebook for Python**, stored as pure Python." `marimo`
replaces the Jupyter `.ipynb` JSON-blob model with `.py` files
where each cell is a function and the runtime tracks the dataflow
DAG between cells. Edit one cell, every dependent cell re-runs;
delete a cell, its variables are GC'd; there is no hidden state and
no stale-cache class of bug. The same file runs three ways: as an
interactive editor (`marimo edit nb.py`), as a deployable web app
(`marimo run nb.py` serves it on `:2718` with no editor surface),
and as a plain CLI script (`python nb.py` or `marimo export script
nb.py`). AI-native by design — every cell has an inline "ask AI"
affordance, the agent can edit cells, run them, and iterate against
your data.

## 1. Install footprint

- `pip install marimo` (Python ≥ 3.9, recommended ≥ 3.11). Optional
  extras: `marimo[recommended]` adds `altair`, `polars`,
  `duckdb`, `openai`, `anthropic`, `google-genai` for the SQL /
  chart / AI-cell surface; `marimo[sql]` for SQL cells via
  `duckdb`; `marimo[lsp]` adds `python-lsp-server` /
  `ruff-lsp` for in-cell type checking and linting.
- Single binary: `marimo` (Typer/click). Subcommands: `edit`, `new`,
  `run`, `tutorial`, `export`, `convert` (Jupyter ↔ marimo),
  `recover`, `check`.
- Standalone WASM build available — `marimo export html-wasm
  nb.py` produces a fully client-side HTML bundle (Pyodide-backed,
  no server) that runs the notebook in a browser, suitable for
  GitHub Pages.
- VS Code extension and a JetBrains plugin published; both shell out
  to the `marimo` binary.

## 2. Repo + version + license

- Repo: <https://github.com/marimo-team/marimo>
- Latest release: **0.23.3** (2026-04-24)
- License: **Apache-2.0** —
  <https://github.com/marimo-team/marimo/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (frontend in TypeScript)

## 3. Models supported

Configurable per-workspace (`~/.config/marimo/marimo.toml` or the
in-app settings panel). First-class providers: OpenAI (chat
completions + Responses API + Azure-style endpoints), Anthropic
(messages, including extended thinking), Google (Gemini AI Studio,
Vertex), and any OpenAI-compatible endpoint (point at a local
`ollama serve`, `mlx_lm.server`, `openllm`, `vllm`, `litellm`
proxy by setting `base_url` + a model id). Bedrock / Ollama have
named providers in the UI. Used for three things: cell generation
("turn this English into a cell"), cell editing ("rewrite this cell
to use polars"), and the agentic chat panel which can read/write
multiple cells across the notebook.

## 4. MCP support

Yes (client, since 0.16). The chat panel supports MCP server
configuration in `marimo.toml` under `[ai.mcp.servers.<name>]` with
`command` + `args` (stdio) or `url` (HTTP). Servers can expose
tools that the marimo agent calls during a chat session — e.g. a
filesystem MCP server lets the agent read CSVs from outside the
notebook directory, a SQL MCP server lets it query an external
warehouse without you wiring credentials through `mo.sql()`. No
server mode (marimo doesn't expose itself as an MCP server).

## 5. Sub-agent model

Single agent, multi-cell scope. The chat panel runs one agent loop
per conversation; it can read all cells in the notebook, propose
edits to one or more cells, run them, observe their outputs (text,
plots, dataframes), and iterate. Tool calls land as cell edits +
runs that you see in the UI, so every step is reversible (`Cmd-Z`
on a cell). No sub-agents, no parallel cell agents — the dataflow
DAG is the parallelism story (cells that don't depend on each other
run concurrently when re-executed). Persistent state lives in cells,
not in the agent; closing the chat panel doesn't lose work.

## 6. Telemetry stance

Opt-in. First-run prompt asks whether to send anonymous usage pings
(cell type counts, feature usage, no cell content); declining sets
`runtime.disable_telemetry = true` in `marimo.toml` and nothing is
sent. AI calls go to whichever provider you configured — your
notebook content (cell sources, recent outputs, agent message
history) is what gets sent on each AI request, not to the
marimo-team servers. The optional `marimo run` deploy mode can
expose a notebook publicly; that's a deliberate `marimo run --host
0.0.0.0` action, not the default.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **No hidden state, ever.** Because cells are a
DAG and not an ordered list, you cannot get into the
"reload-the-kernel-and-pray" hole that Jupyter notebooks fall into
after an hour of out-of-order execution. Rename a variable in cell
A, every cell that read it lights up red until you fix the
reference; delete cell B, its outputs disappear from every
downstream cell; the file on disk is `.py`, so `git diff` is
readable, `pre-commit` works, `pytest` can import it, and your
deploy pipeline doesn't need an `.ipynb` → `.py` shim. Add the
agentic chat panel reading and editing the same cells you're
editing, and you get an AI surface that operates on the same
typed-cell substrate you do — no separate "AI scratch" cell, no
copy-paste back into the notebook. `marimo run` turning the same
file into a hosted app with no rewrite is a nice fourth wall.

**Weakness.** **The reactive model has a learning tax** — you
cannot redefine a variable in a second cell (every name has exactly
one defining cell), `for` loops that mutate cross-cell state are an
anti-pattern, and porting an existing Jupyter notebook usually
requires real refactoring (the `marimo convert` does the syntactic
move but not the dataflow rewrite). Long-running cells block their
downstream subgraph, and there's no first-class "checkpoint /
resume" story for expensive cells (you implement caching with
`@mo.cache` or `functools.lru_cache` yourself). The AI panel is
good for cell-scoped edits but is not a full coding agent — it
won't reorganize the notebook, refactor across files, or run a
shell — and the published notebook ecosystem is much smaller than
Jupyter's, so finding "marimo notebook for X" is hit-and-miss vs.
"Jupyter notebook for X."

**When to choose.** You're building a data app, an interactive
report, an agent eval dashboard, or a teaching surface where the
output is supposed to be reproducible and deployable, not just
"runs on my laptop with this kernel state." Pick `marimo` when you
want a `.py` file your team can review, your CI can lint, and your
ops can deploy — with an interactive editor that doesn't lie about
state. Pair with [`promptfoo`](../promptfoo/) or
[`deepeval`](../deepeval/) when the notebook is an LLM-eval
dashboard, with a local runtime ([`ollama`](../ollama/),
[`mlx-lm`](../mlx-lm/), [`openllm`](../openllm/)) behind the
OpenAI-compatible base URL when you don't want to send notebook
content to a hosted provider. Skip if you need to consume the
massive existing Jupyter `.ipynb` corpus unchanged, if your
notebooks rely on out-of-order interactive exploration as a
feature, or if you want an end-to-end coding agent — marimo's
agent is cell-scoped, not repo-scoped.
