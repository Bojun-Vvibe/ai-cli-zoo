# micro-agent

> Snapshot date: 2026-04. Upstream: <https://github.com/BuilderIO/micro-agent>

A deliberately *small* coding agent. You give it a prompt or point it
at a file, and it generates a **test** first, then iterates on the
implementation until the test passes — capped at a configurable
number of attempts. The slogan is "test-driven AI": the failing test
is the agent's halting condition, not a vibes-based judgement call.

It does not browse the file system, install dependencies, edit
multiple files, or maintain a long plan. The whole project's design
thesis is that general-purpose coding agents fail by *compounding*
errors over many steps; constrain the agent to one file plus one
test runner and the failure mode collapses.

## 1. Install footprint

- `npm install -g @builder.io/micro-agent` (Node ≥ 18).
- Single npm-global install, ~50 MB including dependencies.
- Config persists at `~/.config/@builder.io/micro-agent/` (managed by
  `micro-agent config set ...`).
- API keys live in that config file or as env vars:
  `OPENAI_KEY`, `ANTHROPIC_KEY`, `OPENAI_API_ENDPOINT` (any
  OpenAI-compatible endpoint — Ollama, Groq, vLLM, LiteLLM,
  OpenRouter), `MODEL` (default `gpt-4o`).
- Optional Anthropic key required only if you use the experimental
  `--visual` mode (Claude does the visual matching, OpenAI writes
  the code).

## 2. Repo, version, license

- Repo: <https://github.com/BuilderIO/micro-agent>
- npm package: `@builder.io/micro-agent` (39 release tags as of
  2026-04; project is actively maintained).
- License: MIT. License file at the repo root: `LICENSE`.

## 3. The actual loop

Two run modes:

**(a) Unit-test matching.** Point it at a file plus a test command:

```
micro-agent ./file-to-edit.ts -t "npm test"
```

Default file layout convention:

```
some-folder
├── file-to-edit.ts          # the file the agent is allowed to edit
├── file-to-edit.test.ts     # the test file (override with -f)
└── file-to-edit.prompt.md   # optional extra instructions (override with -p)
```

Loop:

1. If the test file is missing, the agent generates one from the
   prompt.
2. Run `npm test` (or whatever `-t` is).
3. If green: done.
4. If red: rewrite `file-to-edit.ts` (and only that file) given the
   prompt, the current source, and the failing output.
5. Goto 2. Capped at `--max-runs N` (default 10).

**(b) Visual matching (experimental).** `micro-agent ./page.tsx
--visual http://localhost:3000/about`. The agent screenshots the
URL, compares to a `page.png` next to the source, and iterates the
TSX until the rendered output matches. Multi-agent: Claude does the
visual diff, an OpenAI model writes the code.

**(c) Interactive.** Just `micro-agent` with no args drops you into
an interactive REPL that asks what file to edit and what test to
run.

## 4. MCP support

None. It does not consume MCP servers and does not expose one. The
tool surface is "edit one file, run one test command".

## 5. Sub-agent model

In the visual-matching mode, yes: an explicit two-agent pipeline
(Claude critiques, OpenAI writes). In the default unit-test mode,
no — single agent, single file scope.

## 6. Telemetry stance

Off (no analytics in the published code as of v` *check `package.json`*).
The provider endpoint you configure sees prompts that include the
file source, the test file, the test output, and any `prompt.md`
contents.

## 7. Token / context strategy

Per iteration the prompt is roughly: system instructions + the
current source file + the test file + the most recent test output.
Bounded by the file's own size — there is no whole-repo packing,
which is the design point. Long files near the model's context limit
will fail; the recommended pattern is to scope the agent to one
file at a time and use a real coding agent
([`aider`](../aider/), [`claude-code`](../claude-code/)) when you
need multi-file refactors.

`--max-runs N` (default 10) is the cost cap. Each run is one full
prompt + one full code rewrite. Budget accordingly when the model is
slow or expensive.

## 8. Hot keybinds

In interactive mode (`micro-agent` with no args):

- Arrow keys / `Enter` to navigate the prompts (CLI uses
  `@clack/prompts`).
- `Ctrl-C` aborts the current run.

In file-mode there is no TUI; output is streamed line by line and
each iteration prints a `[run N/M]` header.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The failing test is the halting condition.**
Most catalog agents (`opencode`, `claude-code`, `cline`, `OpenHands`,
`aider`) decide on their own when they're "done", which is a major
class of bugs (early-exit, hallucinated success, wrong files
touched). `micro-agent` defers that decision to a real test runner
you already trust — the same npm script your CI runs. If the test
goes green, the agent stops; if it fails 10 times, the agent stops
*and tells you it failed*. Determinism as a feature.

**Weakness.** Single-file scope is a hard ceiling. You cannot ask
it to "add a new endpoint" if that endpoint requires touching the
router, the controller, and the OpenAPI spec. The visual-matching
mode is explicitly experimental and requires both an Anthropic and
an OpenAI key. No MCP, no plugins, no model-routing beyond
"OpenAI-compatible or Anthropic". The CLI is intentionally minimal,
which means it has no escape hatches when you outgrow it — you just
switch tools.

**When to choose.**
- You're writing a **pure function** or a **small leaf module** with
  a clear test contract.
- You **already have a test command** (`npm test`, `pytest`, `cargo
  test` invoked through a wrapper) and want the agent's loop bounded
  by it.
- You want a coding agent whose **stop condition is observable in
  CI**, not "the model said it's done".
- You're learning TDD and want a tool that enforces "test first,
  then code".

**When to skip.**
- The change spans **multiple files** or the test setup itself needs
  edits — use [`aider`](../aider/), [`claude-code`](../claude-code/),
  or [`opencode`](../opencode/).
- You want a **plan-then-execute** flow over many steps —
  [`plandex`](../plandex/) or [`OpenHands`](../openhands/).
- You want an agent that can **install dependencies** or **scaffold
  a project from scratch** — [`smol-developer`](../smol-developer/)
  or a sandboxed agent like [`codex`](../codex/).
- You want **structured-output Python embedding** of LLM calls
  rather than a CLI loop — [`magentic`](../magentic/).

## 10. Compared to neighbors in the catalog

| Tool | Scope | Halt condition | Multi-file | Sandboxing | Lang |
|------|-------|----------------|------------|------------|------|
| micro-agent | One file + one test command | Test passes (or N retries) | No | Inherits your shell | TS (Node) |
| [aider](../aider/) | Whole repo via repo-map | Model says done; you `y/n` each diff | Yes | None | Python |
| [claude-code](../claude-code/) | Whole repo + tools + sub-agents | Model says done; hooks/skills can gate | Yes | None (host shell) | TS (Node) |
| [plandex](../plandex/) | Whole repo with branching plans | Plan completes | Yes | Sandbox via worktrees | Go |
| [smol-developer](../smol-developer/) | Greenfield — generates a whole repo from one paragraph | Generation completes | Yes | None | Python |
| [openhands](../openhands/) | Whole repo + browser + Linux sandbox | Model says done; multi-agent vote | Yes | Full Linux sandbox | Python |

Decision shortcut:

- "TDD this one function until the test passes" → `micro-agent`.
- "Edit a few related files with `y/n` per diff" → `aider`.
- "Long-horizon multi-step plan with branching" → `plandex`.
- "Greenfield: produce a repo from a paragraph" → `smol-developer`.
- "Run agents in a real Linux sandbox" → `openhands`.
