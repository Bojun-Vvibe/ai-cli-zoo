# probe

> Snapshot date: 2026-04. Upstream: <https://github.com/buger/probe>
> License file: <https://github.com/buger/probe/blob/main/LICENSE>
> (`LICENSE`, sha `e4f139cb498a107999732232802ca3e737d8d212`,
> 10 254 bytes).
> Pinned: `v0.6.0-rc315` (latest release, 2026-04-06).
> HEAD `045bbca` on `main`. Apache-2.0 throughout.

An **AI-friendly semantic code-search engine** built for one job:
when an LLM agent asks "show me the code that handles X", return
*tree-sitter-bounded* snippets — whole functions, classes, methods —
not the broken-mid-line context that a raw `ripgrep` would give.
Probe combines `ripgrep`-class regex speed (it links the
ripgrep matcher) with tree-sitter parsers for ~25 languages,
so a hit inside `function foo() { ... if (cond) { /* match */ } }`
returns the whole `foo` body, not three lines around the match.
Ships as a single Go binary plus an MCP server, an OpenAI
function-calling adapter, and a small Node SDK — designed to be
the "code search tool" that agent frameworks plug in.

## 1. Install footprint

- **Single binary**: `curl -fsSL
  https://raw.githubusercontent.com/buger/probe/main/install.sh |
  bash` — installs `probe` to `~/.local/bin` (~25 MB statically
  linked Go binary, no runtime deps, no Python, no Node).
- **Homebrew**: `brew install buger/probe/probe`.
- **From source**: `cargo install` is *not* the path — the binary
  is Go; the `ripgrep`-derived matcher is vendored. `go install
  github.com/buger/probe/cmd/probe@latest`.
- **MCP server**: bundled subcommand `probe mcp` — speaks
  stdio MCP, no extra install. Add to an MCP client config
  pointing at the binary.
- **Node SDK**: `npm install @buger/probe` — wraps the binary
  for Node-native agent frameworks; the binary is downloaded on
  postinstall.
- Workspace footprint: zero by default — probe is stateless and
  re-parses on every search. Optional `--cache` flag persists
  parsed ASTs to `~/.cache/probe/` for faster repeat runs on
  large monorepos (~10× speed-up on a 1 GB tree).

## 2. Repo, version, license

- Repo: <https://github.com/buger/probe>
- Default branch: `main` (HEAD `045bbca` at snapshot).
- Latest release: `v0.6.0-rc315` (2026-04-06). The project
  ships frequent release-candidates with semver-shaped tags;
  `-rc<n>` is the production-recommended track per the
  release notes.
- License: **Apache-2.0**. License file: `LICENSE`
  (sha `e4f139cb498a107999732232802ca3e737d8d212`, 10 254
  bytes). Author Leonid Bugaev (`buger`, of `goreplay` fame)
  maintains it; no CLA, no enterprise tier.

## 3. What it actually does

Three call shapes:

1. **`probe search "<query>" [<path>]`** — regex *or*
   natural-language query. With `--natural-language` (default
   on for multi-word queries) probe does light query expansion
   (synonyms, stemming, language-aware tokenisation). Output is
   ranked code blocks, each one a *complete* tree-sitter node
   (function / class / method / interface) containing the
   match, with file path + line range.
2. **`probe extract <file>:<line>`** — given a file and line
   number, return the smallest enclosing semantic block. The
   primitive an agent uses after a regex search to expand
   "match on line 87" into "the function that contains line 87".
3. **`probe query <tree-sitter-pattern>`** — direct AST queries
   in tree-sitter S-expression syntax. `(function_declaration
   name: (identifier) @name (#match? @name "^handle"))` returns
   every Go function whose name starts with `handle`.

Supported languages (~25): Go, Rust, Python, JS, TS, TSX, Java,
C, C++, C#, Ruby, PHP, Swift, Kotlin, Scala, Elixir, Erlang,
Lua, Bash, Markdown, JSON, YAML, HTML, CSS. Detection is by
file extension; mixed-language repos work without configuration.

## 4. MCP support

**First-class MCP server**, bundled. `probe mcp` exposes three
tools: `search_code` (natural language or regex, returns ranked
semantic blocks), `query_code` (tree-sitter S-expression),
`extract_code` (file:line → smallest enclosing block). Every
tool returns code with file path + line range so the agent can
follow up with a precise edit.

This is the integration shape the project optimises for: the
README shows configs for [`claude-code`](../claude-code/),
[`opencode`](../opencode/), [`cline`](../cline/), [`crush`](../crush/),
[`continue`](../continue/), [`fast-agent`](../fast-agent/), and
generic stdio MCP clients. Compared to giving an agent raw
`grep` / `ripgrep`, the win is "no broken-mid-function snippets
in the agent's context" — every returned block is parseable on
its own, which dramatically reduces follow-up "show me more
context around line 87" round-trips.

## 5. Sub-agent model

None — probe is a tool, not an agent. It is the *thing an
agent calls*. The closest internal concept is the parallel
worker pool that fans out tree-sitter parsing across CPU
cores during a single search; that is invisible to the caller
and just makes a search of a 100 K-file repo finish in
sub-second.

For "agent that searches code and edits it", probe is the
search half; pair with [`aider`](../aider/),
[`opencode`](../opencode/), or [`claude-code`](../claude-code/)
as the edit half. The MCP integration makes the pair zero-glue.

## 6. Telemetry stance

**No telemetry.** The binary makes no outbound network calls
during search; everything is local file IO + CPU. The
project's only network surface is the install script's
download from GitHub releases. Air-gapped use is the default
and requires no flags. Compare to [`bloop`](../bloop/) (which
historically required a cloud account) and
[`seagoat`](../seagoat/) (which calls an embedding API) —
probe is the lowest-egress code-search option in this corner
of the catalog because it does *no embeddings*; ranking is
lexical / structural.

## 7. Token / context strategy

The whole point. Default search returns the top *N* semantic
blocks ranked by `--max-results` (default 10) and capped by
`--max-tokens` (default 40 000) so an agent can confidently
ask "search the codebase for X" without blowing its context.
Each block is a complete tree-sitter node, so the agent never
sees a half-function. `--max-tokens` budget is enforced
*before* serialisation: if the top blocks would exceed it,
probe returns fewer, longer blocks rather than truncating any
single block — preserving the "every snippet is parseable"
invariant.

The `extract` subcommand follows the same rule: ask for
`file.go:87` and you get the *smallest* enclosing semantic
block (innermost function / method) by default; pass
`--max-tokens` to expand outward to the enclosing class /
file as the budget allows.

## 8. Hot keybinds

No TUI. Everyday surface is the CLI:

```bash
# 1. Natural-language search across the repo (top 10 semantic blocks)
probe search "where do we validate the JWT signature"

# 2. Regex search, capped at 20 K tokens of output
probe search --pattern "func.*Handler" --max-tokens 20000 ./internal

# 3. Tree-sitter AST query — every Python class whose name ends in "Service"
probe query '(class_definition name: (identifier) @n (#match? @n "Service$"))' ./src

# 4. Expand a single line into its enclosing function
probe extract internal/auth/jwt.go:87

# 5. As an MCP server (stdio); wire into your agent client config
probe mcp
```

Useful flags worth knowing:

- `--language <lang>` — restrict to one tree-sitter parser
  (faster on heterogeneous trees).
- `--max-tokens <n>` — hard cap on total output, with
  block-preserving truncation.
- `--max-results <n>` — cap on number of blocks returned.
- `--exclude <glob>` — standard `.gitignore`-style exclusion
  on top of the repo's own ignore files.
- `--cache` — persist parsed ASTs in `~/.cache/probe/` for
  ~10× speed-up on repeat searches over the same tree.
- `--format json` — machine-readable output for SDK / pipeline
  consumption.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Search results an LLM can actually
read.** Every hit is a *complete* tree-sitter block (function,
class, method, interface), capped to a token budget the agent
declares up front. No half-functions, no broken context, no
"show me more around line 87" follow-up round-trips. The MCP
server ships in the same binary, so wiring it into
[`claude-code`](../claude-code/), [`opencode`](../opencode/),
[`cline`](../cline/), or [`crush`](../crush/) is a one-line
config change, not a separate process to babysit.

**Weakness.** **Lexical / structural ranking, not semantic
embedding ranking.** "Find code with the same *idea* as this
function" is a job for [`seagoat`](../seagoat/) or
[`bloop`](../bloop/), not probe. **Pre-1.0 with `-rc` tags** —
the API is mostly stable but the project ships fast; pin a
specific tag for CI. **Tree-sitter coverage** is wide but not
universal — exotic languages without a maintained tree-sitter
grammar fall back to plain regex (still fast, but you lose the
"complete block" guarantee).

**When to choose.**

- You are wiring **a code-search tool into an LLM agent** and
  want the snippets back to *fit a token budget* and *parse
  cleanly on their own*.
- You want **MCP-native code search** with zero glue —
  `probe mcp` and you are done.
- You want a **single static binary** with no Python / Node
  runtime and no daemon to keep alive.
- You want **zero network egress** from the search tool —
  no embedding API calls, no telemetry, no cloud.
- You need **cross-language coverage** (Go + TS + Python +
  Rust monorepos) without configuring per-language indexers.

**When not to choose.** You want **semantic "find code that
does the same thing" search** powered by embeddings → pick
[`seagoat`](../seagoat/) or [`bloop`](../bloop/). You only
need **structural AST search inside one language** without
the LLM angle → use [`ast-grep`](../ast-grep/) directly. You
want a **codebase-aware chat UI** rather than a search tool
an agent calls → pick [`bloop`](../bloop/) or
[`aider`](../aider/) with its built-in repo-map. You need a
**hosted, indexed code-search service** for a 50-engineer
team across many private repos → pick [`bloop`](../bloop/) or
a managed Sourcegraph; probe is for the local / per-repo / per-
agent case.
