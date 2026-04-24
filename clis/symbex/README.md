# symbex

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/symbex>
> Binary name: `symbex`

`symbex` is the **surgical symbol extractor** of the catalog. Where
[`files-to-prompt`](../files-to-prompt/) packs whole files (and
respects `.gitignore` while doing it), `symbex` operates one level
deeper: it parses Python source with the standard-library `ast`
module and emits **only the named symbols you asked for** — a single
function body, a class with all its methods, every signature in a
module, every method whose name matches a glob.

That single design decision is what makes `symbex` distinct from the
generic context-packers in this catalog. The output of
`symbex 'MyClass.*'` is the literal source of every method on
`MyClass`, with file/line annotations as comments, suitable for
piping straight into `llm`, `mods`, `claude-code -p`, or any other
catalog entry. The shape is "minimum tokens to answer this
question", not "everything that might possibly be relevant".

Three usage shapes define the surface:

- `symbex my_function` — print the source of `my_function` from
  anywhere under the current directory.
- `symbex 'MyClass.*'` — print every method on `MyClass`.
- `symbex --signatures --imports 'pkg/'` — print only signatures
  (no bodies) plus the matching `from x import y` lines, for a
  cheap module-shape overview.

There are also `--documented` / `--undocumented` (filter by
docstring presence), `--async`, `--function`, `--class`,
`--unwrap` (strip decorators from the output), and `--silent`
(skip files that fail to parse instead of erroring out).

## 1. Install footprint

- Pure Python, single-file CLI, ~50 KB on disk plus the standard
  `click` dependency. No native deps, no model SDKs.
- `pipx install symbex` (recommended), `uv tool install symbex`, or
  `pip install symbex` inside a venv.
- Zero runtime config. There is no config file, no API key, no
  network call. The only state is the AST it builds in memory per
  invocation.
- Works on any Python source the standard-library `ast` module
  accepts, which in practice means Python 3.8+ syntax. Files that
  fail to parse are reported on stderr; `--silent` suppresses that.

## 2. License

Apache-2.0.

## 3. Models supported

**None.** `symbex` is not a model client. It does not call OpenAI,
Anthropic, Gemini, Ollama, or anything else. It reads `.py` files,
walks the AST, and writes matching source to stdout.

This is the same posture as [`files-to-prompt`](../files-to-prompt/):
it produces context for *some other* CLI in the catalog to consume.
The intended composition is `symbex 'MyClass.*' | llm -m
claude-sonnet 'rewrite this class to be thread-safe'` or
`symbex --signatures 'mypkg/' | mods 'list every public function'`.

## 4. MCP support

None. There is no MCP client and no MCP server. `symbex` is a Unix
filter — text in (file paths and symbol globs), text out (matching
source). If you want MCP-style tool calling that *uses* `symbex`,
wrap it in a script and expose it as an MCP server yourself; the
upstream README explicitly endorses that pattern.

## 5. Sub-agent model

None. There is no agent loop, no LLM call, no recursion. One
invocation parses N files and prints M matching symbols, then
exits. Compose with shell pipes, not with internal orchestration.

## 6. Telemetry stance

**Off, with no opt-in.** No network calls of any kind. The binary
opens local files, walks ASTs in memory, and writes to stdout. This
is the same "no-network" posture as `files-to-prompt`, and it is
exactly what you want in an air-gapped or pre-commit-hook context.

## 7. Prompt-cache strategy

Not applicable — `symbex` does not call any model and has no notion
of a prompt. Its *output* is what your downstream LLM CLI will
cache or not cache; if the downstream client supports prompt-prefix
caching (Anthropic, Gemini), keeping `symbex` invocations
deterministic (same args → byte-identical output) is the relevant
property, and it satisfies that.

## 8. Hot keybinds

There is no TUI and no REPL — `symbex` is a one-shot CLI. The
ergonomic surface is the flag set, and the most useful shapes are:

- `symbex 'MyClass.method_name'` — fully qualified, single method.
- `symbex 'MyClass.*'` — every method on a class.
- `symbex '*.handle_*'` — every method named `handle_*` on any class.
- `symbex --signatures 'pkg/'` — module-shape overview, no bodies.
- `symbex --documented --signatures 'pkg/'` — only public-API
  candidates (those that bothered to write a docstring).
- `symbex --no-init --no-pyi 'pkg/'` — skip `__init__.py` and stub
  files, useful when packing a real-implementation overview.
- `symbex --imports 'pkg/'` — emit the `import` lines too, so the
  downstream LLM can resolve names without you handing it the whole
  module.

The `-f file.py` form scopes a query to a specific file, and
`--directory pkg/` scopes it to a subtree (overriding the default
"recursively scan from `.`" behavior).

## 9. Killer feature, weakness, when to choose

**Killer feature.** Token-efficient, AST-correct context. If your
question is "rewrite this one method", you do not need to send the
entire 4,000-line module — `symbex MyClass.handle_request` gives the
model exactly the function body, and nothing else. For codebases
where you are paying per-token (or fighting a context window),
`symbex` regularly cuts context size by an order of magnitude versus
`files-to-prompt path/to/module.py`.

**Weakness.** Three:

1. **Python only.** The AST parser is Python's `ast` module; there
   is no TypeScript, Go, Rust, or Java story. For a polyglot repo
   you fall back to `files-to-prompt` (whole files) or to language-
   specific tools (`tree-sitter`-based, etc.).
2. **No semantic resolution.** `symbex` matches symbol *names*, not
   *call sites*. "Show me everything that calls `MyClass.foo`" is
   not in scope — that needs an LSP-backed tool, not an AST walker.
3. **Glob-only matching.** No regex on bodies, no "methods longer
   than 50 lines", no "methods that mention `requests`". Filtering
   is by name, decorator, async/sync, documented/undocumented, and
   class/function — that is the entire knob set.

**When to choose.**

- **You are about to paste a 2,000-line file into a chat to ask
  about one method.** Run `symbex MyClass.that_method` and pipe it
  to your LLM CLI of choice. You will get a better answer for
  fewer tokens.
- **You want to brief an LLM on a Python package's public surface.**
  `symbex --signatures --documented --imports 'pkg/'` produces a
  module-shape overview that fits in 4 KB for most packages.
- **You are building a custom prompt template** that needs "the
  source of these specific functions". `symbex` is the deterministic
  primitive; any other approach (grep + sed + cut) breaks on
  multi-line decorators, nested classes, and triple-quoted strings.
- **Pre-commit / CI context generation.** No network, no key, no
  flakiness — runs in a sandbox or air-gapped runner without
  configuration.

**When not to choose.**

- **Your codebase is not Python.** Reach for `files-to-prompt`
  (language-agnostic, file-level) or a tree-sitter-based tool.
- **You want the model to *find* the relevant symbol itself.**
  That is what `aider`'s repo-map, `claude-code`'s `Grep`/`Glob`
  tools, and `opencode`'s file tools are for — they let the agent
  navigate. `symbex` assumes you already know the symbol name.
- **You want a chat-style REPL.** `symbex` is a Unix filter and
  nothing else; pair it with [`llm`](../llm/), [`mods`](../mods/),
  or [`shell-gpt`](../shell-gpt/).
- **You need cross-file call-graph analysis.** That is an LSP /
  static-analysis problem, not an AST-extraction one.
