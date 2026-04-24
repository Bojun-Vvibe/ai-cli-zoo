# serena

> Snapshot date: 2026-04. Upstream: <https://github.com/oraios/serena>

"**Serena is the IDE for your coding agent.**" A semantic-code-retrieval
and editing toolkit that runs as an MCP server, exposing **symbol-level**
operations (find symbol, find references, replace symbol body, insert
before/after symbol) backed by real Language Server Protocol (LSP)
servers. The pitch is that any MCP-capable coding agent (`opencode`,
`codex`, `claude-code`, `crush`, Cursor, Cline) gets IDE-grade
cross-file rename / reference-lookup / symbolic edit primitives instead
of the line-number text surgery that line-and-regex tools force.

## 1. Install footprint

- Recommended: `uvx --from git+https://github.com/oraios/serena serena start-mcp-server`.
- Also supported: `pipx install --pip-args="--prerelease=allow" serena-agent`,
  Docker image, native binary downloads in releases.
- Bundles per-language LSP launch logic for Python (Pyright /
  pyright-langserver), TypeScript / JavaScript, Go (gopls), Rust
  (rust-analyzer), Java (Eclipse JDT), C# (omnisharp), C / C++ (clangd),
  Ruby (Solargraph), PHP (Intelephense), Kotlin, Dart, Lua, Bash, Elixir,
  Clojure, Terraform, Zig, Nix, Swift, Erlang. First run on a project
  pulls / launches the matching LSP automatically.
- Project state lives under `.serena/` in the target repo (memory files,
  cache).

## 2. Repo + version + license

- Repo: <https://github.com/oraios/serena>
- Latest release: **v1.1.2**
- License: **MIT** —
  <https://github.com/oraios/serena/blob/main/LICENSE>
- Default branch: `main`

## 3. Models supported

None directly. Serena is an **MCP server**: it ships tools, not a model
client. The model lives in whatever MCP client you point at it
(Anthropic / OpenAI / Gemini / local — whatever your `opencode` /
`codex` / `claude-code` / Cursor session is configured for).

## 4. MCP support

**Serena is an MCP server, top to bottom.** That is the entire surface.
`serena start-mcp-server` speaks stdio MCP (or HTTP with a flag) and
publishes tools like `find_symbol`, `find_referencing_symbols`,
`replace_symbol_body`, `insert_after_symbol`, `read_memory`,
`write_memory`, plus a `dashboard` tool. There is no chat REPL of its
own.

## 5. Sub-agent model

None. Serena is the *toolbelt* an external agent uses; sub-agency lives
in the MCP client (e.g. `claude-code`'s `Task` tool, `opencode`'s
`Task`).

## 6. Telemetry stance

Off. No analytics in the OSS codebase. Egress is exactly the LSP
processes it spawns (all local) and whatever MCP client connects to it.
The optional dashboard binds `127.0.0.1` only.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **LSP-backed symbolic edits exposed as MCP tools.**
Where a generic agent would read a 2 000-line file, find a function with
a regex, and try to rewrite it without breaking the surrounding braces,
Serena's `replace_symbol_body` operates on the parsed AST via the
language server: the agent says "replace the body of
`UserService.authenticate`" and either it succeeds atomically or it
fails cleanly. Cross-file rename and reference lookup are similarly
single-call, language-correct operations rather than multi-step text
patches. The published end-user evaluation (run by the agents
themselves) consistently puts Serena's tools above the host agent's
built-in file-grep workflow on multi-file refactors.

**Weakness.** Adds an LSP process per language to your machine — the
first run on a fresh repo is slow as the language server indexes, and
LSPs for large polyglot monorepos can be RAM-heavy. There is no
stand-alone CLI: if your editor / agent does not speak MCP, Serena is
not for you. Memory files (`.serena/memories/`) live in-repo and need
to be `.gitignore`d for solo workflows or committed deliberately for
team workflows — there is no opinion baked in.

**When to choose.** You already use an MCP-capable coding agent on a
non-trivial codebase, you have noticed it wasting tokens reading whole
files to perform what should be one-symbol edits, and you want
language-server-grade correctness on rename / move / reference-lookup
without leaving the terminal. Pair with the catalog's other MCP
servers (`repomix --mcp` for whole-repo packing, `kubectl-ai
--mcp-server` for cluster operations, `container-use` for sandboxed
parallel agents). Skip if your stack is a small single-file project
where regex grep is already fine, or if your agent is not MCP-capable.
