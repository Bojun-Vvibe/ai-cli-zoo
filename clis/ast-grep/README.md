# ast-grep

> Snapshot date: 2026-04. Upstream: <https://github.com/ast-grep/ast-grep>
> License file: <https://github.com/ast-grep/ast-grep/blob/main/LICENSE>
> Pinned: `0.42.1` (2026-04-04). Releases land monthly; the
> pattern-language and YAML rule schema are stable, but
> tree-sitter grammar versions ship with each release and a rule
> that matched on `0.40.x` may match more (or fewer) nodes on
> `0.42.x`.

A **CLI for code structural search, lint, and rewriting** that
matches **AST nodes**, not text. Pattern-language is "code with
metavariables" — `var $X = $Y` matches every `var` declaration in
JS/TS regardless of the actual identifier and initialiser, and you
can rewrite to `let $X = $Y` to flip the whole codebase in one
command. Built on tree-sitter, written in Rust, single static binary.

Not an LLM CLI itself, and that is exactly why it belongs in this
catalog: it is the **deterministic, model-free counterpart to the
"have an agent refactor my codebase" workflow**. When you can express
a refactor as a syntactic pattern, ast-grep is faster, cheaper, and
correct by construction; when you can't, you reach for an LLM-driven
agent like [`aider`](../aider/) or [`opencode`](../opencode/). It is
also the right tool to **hand to an LLM agent as an MCP tool** for
the structural part of a refactor — the agent decides the policy,
ast-grep enforces the substitution.

## 1. Install footprint

- `npm install --global @ast-grep/cli`, `pip install ast-grep-cli`,
  `cargo install ast-grep --locked`, `brew install ast-grep`,
  `scoop install main/ast-grep`, `port install ast-grep`,
  `nix-shell -p ast-grep`, or `mise use -g ast-grep` — pick whichever
  packager you already use.
- Single static binary on Linux, macOS, Windows; x86_64 and arm64. No
  runtime, no daemon, no project state.
- Workspace footprint: zero by default. With YAML rule mode you keep
  rules in `sgconfig.yml` + a `rules/` directory and that's it.
- Two entry points in the same binary: `ast-grep` and the shorter
  alias `sg` (which conflicts with `super-grep` in some distros —
  the `ast-grep` name is the safe one for scripts).

## 2. License

MIT.

## 3. Models supported

**None.** ast-grep is purely deterministic; there is no LLM in the
loop, no embeddings, no semantic search. Pattern matching is exact
syntactic match against the tree-sitter parse tree of the target
language.

Languages it supports out of the box include JavaScript, TypeScript,
TSX, Python, Rust, Go, Java, Kotlin, Swift, C, C++, C#, Ruby,
PHP, Lua, Bash, HTML, CSS, JSON, YAML, Solidity, and more — anything
with a tree-sitter grammar shipped in the binary. Custom languages
can be loaded via `customLanguages` in `sgconfig.yml`.

## 4. MCP support

**Yes — via [`ast-grep-mcp`](https://github.com/ast-grep/ast-grep-mcp),
a separate package.** The headline use case: hand ast-grep to an LLM
coding agent as a structured search/replace tool, so the agent stops
trying to do regex-style edits and instead emits an ast-grep pattern
+ rewrite. This dramatically reduces both token cost and the rate of
"agent broke a totally unrelated identifier because the regex was too
greedy" failures. Wire it into [`opencode`](../opencode/),
[`claude-code`](../claude-code/), [`goose`](../goose/), or any other
MCP client.

## 5. Sub-agent model

None. One pattern, one pass over the file tree, one result set. The
parallelism story is "Rust + Rayon across files," not agent
orchestration.

## 6. Telemetry stance

**Off, no opt-in.** No analytics; no network calls during normal
operation. The only egress is what your installer does (npm registry,
PyPI, crates.io, Homebrew bottle host) on `install`. After that the
binary is fully offline.

## 7. Prompt-cache strategy

N/A — no LLM. The relevant cache is **the tree-sitter parse cache**:
ast-grep parses each file once per invocation and re-uses the tree
across all rules in the same run. For huge repos, batching rules into
a single `ast-grep scan` invocation is therefore meaningfully faster
than running `ast-grep run` once per rule.

## 8. Hot keybinds

No TUI in the default mode (there is an `--interactive` review flow
for rewrites). Three operating modes:

- **`run` mode — one-off pattern search/replace from the command
  line.**
  - `ast-grep run -p 'console.log($A)' -l ts` — find every
    `console.log(...)` in TypeScript.
  - `ast-grep run -p '$A && $A()' -l ts -r '$A?.()'` — rewrite
    short-circuit calls to optional-chaining calls.
  - `ast-grep run -p 'var $X = $Y' -r 'let $X = $Y' -l js -i` —
    `-i` opens an interactive accept/reject prompt per match.
  - `ast-grep run -p 'fn $F($$$)' -l rust` — `$$$` matches zero or
    more nodes (varargs / variadic lists).
- **`scan` mode — run a directory of YAML rules like a linter.**
  - `ast-grep scan` — apply every rule in `sgconfig.yml`.
  - `ast-grep scan --filter 'no-console-log'` — single rule.
  - `ast-grep scan --report-style rich` — pretty CLI report;
    `--json` for machine-readable, `--format github` for GitHub
    Actions annotations.
- **`new` / `test` mode — author and unit-test rules.**
  - `ast-grep new rule` — scaffold a new YAML rule.
  - `ast-grep test` — run the rule's `test/valid.ts` /
    `test/invalid.ts` cases like a parser test suite.

YAML rule shape (excerpt — the killer feature is composability):

```yaml
id: no-await-in-loop
language: TypeScript
rule:
  pattern: await $X
  inside:
    any:
      - kind: for_statement
      - kind: while_statement
      - kind: for_in_statement
fix: Promise.all($X)
message: Avoid `await` inside loops; batch with Promise.all.
severity: warning
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The pattern language is isomorphic to the
target language**, and metavariables (`$X`, `$$$`) sit in exactly
the place a real identifier would. That makes ast-grep patterns
**readable by anyone who can read the source language**, unlike
Comby templates, codemods written as Babel plugins, or
sed-of-doom one-liners. Combine that with **YAML rules** (composable
`all` / `any` / `not` / `inside` / `has` / `follows` constraints, with
fixes), the **single static Rust binary** (so CI integration is `npm
i -g @ast-grep/cli && ast-grep scan`), and the **MCP server** (so an
LLM agent can use it as a precise refactor tool instead of doing
regex edits), and you have the modern replacement for
`grep | xargs sed` across multi-language repos.

**Weakness.** **Pattern compilation is exact-syntactic**, so
`foo.bar` and `foo['bar']` are two different patterns even though
they are semantically equivalent — you write both, or write a
broader `kind:` rule. **No type information** (tree-sitter is a
parser, not a type-checker), so refactors that depend on "is this an
instance of class X" are out of scope; reach for `ts-morph`,
`jscodeshift`, `gritql`, or a real LSP-based tool. The **`sg` binary
name conflicts** with `super-grep` on some Linux distros; the safer
script-friendly name is `ast-grep`. And the YAML rule schema, while
expressive, has a learning curve roughly equal to "write your first
ESLint plugin from scratch."

**When to choose.**

- You have a **large multi-language repo** and you want to enforce
  syntactic conventions (no `console.log`, no `var`, no
  `Promise.then` chains in async functions, etc.) without writing N
  ESLint / Pylint / Clippy plugins.
- You need a **codemod** that's larger than a sed pipeline but
  smaller than a full `jscodeshift` script — ast-grep `run -p ... -r
  ... -i` is the right granularity.
- You are wiring a **coding agent**
  ([`opencode`](../opencode/), [`claude-code`](../claude-code/),
  [`goose`](../goose/)) and want to give it a **deterministic
  search/replace tool over MCP** so it stops eating tokens on
  greedy regex edits.
- You want **pre-commit lint rules** that one of your linters can't
  express because it requires positional context (e.g. "no `console`
  inside a loop, but it's fine at module top-level").

**When not to choose.** Your refactor needs **type information** or
**cross-file symbol resolution** — use a TS-Morph / Roslyn /
`rust-analyzer` driven script. The change can be expressed as a
short shell pipeline — `ripgrep` + `sd` is fewer dependencies. The
change is **inherently semantic** ("rename this variable everywhere
it's used, including indirect references") — that is an LSP /
language-server job, not a syntactic-match one. Or the work is
better described in English than in a pattern — then you've earned
your way to an LLM-driven agent like [`aider`](../aider/) or
[`opencode`](../opencode/).
