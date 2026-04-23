# ARCHITECTURE-PATTERNS.md

A cross-cutting look at how these 12 CLIs are built. The interesting question
isn't "which model do they call" — it's "what loop wraps the model call, what
context do they feed it, and what guardrails do they put around tool use."

## 1. The agent loop, four flavors

Every CLI here implements one of four loops.

### 1a. Diff-confirm loop (aider, mentat)

```
user prompt
   ↓
build context (repo-map / open files)
   ↓
LLM → diff/edit-block
   ↓
show diff → user approves → git apply → git commit
```

- **Pro:** trivially auditable. Every change is a diff you saw.
- **Con:** no autonomous multi-step work. The model can't "try, observe, retry".

### 1b. Tool-call loop with approval (cline, opencode default, continue agent mode)

```
user prompt
   ↓
LLM → tool call (read/write/bash/...)
   ↓
[approve?]  ── no → bail
   ↓ yes
execute → result → back to LLM
   ↓
... until LLM emits "done"
```

- **Pro:** model can iterate (read → think → write → re-read).
- **Con:** approval fatigue. Every CLI in this category eventually adds
  "auto-approve allowlist" to address this.

### 1c. Sandboxed autonomous loop (codex, OpenHands)

```
user prompt
   ↓
spawn sandbox (Seatbelt / Landlock / Docker)
   ↓
LLM ↔ tool calls executed inside sandbox
   ↓
diff vs. host filesystem on exit → user reviews
```

- **Pro:** model is fully autonomous, host is safe.
- **Con:** sandbox setup is OS-specific; Linux has `landlock`/`bubblewrap`,
  macOS has Seatbelt, Windows has nothing comparable so most ship a Docker
  fallback.

### 1d. Plan-then-execute (plandex, sweep, smol-developer)

```
user prompt / issue
   ↓
"planner" LLM call → ordered task list
   ↓
"builder" LLM calls one task at a time
   ↓
optional review LLM call → emit PR / final tree
```

- **Pro:** scales to multi-hour work. State is checkpointable.
- **Con:** if the plan is wrong the whole run is wrong; rollback is awkward.

## 2. Context strategies

| Strategy | Used by | How it works |
|----------|---------|--------------|
| Repo-map (tree-sitter) | aider | Parse every source file with tree-sitter, extract symbols, rank by PageRank-ish, fit top-N into context. |
| File-list + read tool | opencode, codex, cline, crush, continue | Don't preload. Let the model call `read` / `glob` / `grep` tools as it needs. |
| RAG over chunks | continue (optional), OpenHands | Embed the repo, retrieve top-k chunks per query. |
| Single-file scope | mentat, gptme | Only files the user explicitly added enter context. |
| Issue-as-context | sweep | The GitHub issue body + linked code is the entire seed. |
| Whole-tree dump | smol-developer | Everything, since "everything" is small in greenfield mode. |

The trend is clear: **read tools beat preloaded context** once models got good
at calling them. Aider's repo-map is the holdout, and it's still the best
choice for "I have a 10k-file repo and I want one focused edit."

## 3. Tool-call shape

Three formats in the wild:

1. **OpenAI function-calling JSON** — codex, continue, smol-developer.
2. **Anthropic XML-ish tool blocks** — cline, opencode, crush, OpenHands.
3. **Markdown edit-blocks** (no native tool calls) — aider, mentat.
   The model emits ` ```diff ` or aider's `SEARCH/REPLACE` blocks; the CLI
   parses them out.

Edit-block parsing is older and clunkier but works against any model,
including ones with no tool-call training. That's why aider can target Ollama
models that fail at proper JSON tool calls.

## 4. Sub-agent patterns

Most CLIs here are single-agent. The exceptions:

- **opencode** has a `Task` tool the main model can call to spawn a
  sub-agent with its own context window. Pattern: "use a sub-agent to grep
  the codebase, return only the answer." Saves tokens.
- **OpenHands** runs distinct *agent classes* (CodeActAgent, BrowsingAgent,
  etc.) and routes tasks between them. More like an OS scheduler than a
  call tree.
- **plandex** has an internal planner / builder split — same model, different
  system prompts and contexts.
- **sweep** is technically multi-stage (search → plan → write → review) but
  each stage is a fresh LLM call, no shared agent state.

## 5. Sandboxing & approval

Ranked from most to least restrictive *out of the box*:

1. **codex** — Seatbelt sandbox (macOS) / Landlock (Linux). Read-only
   filesystem outside the project dir, no network.
2. **OpenHands** — Docker container by default. Full Linux + browser inside.
3. **opencode** — no sandbox; per-tool approval. Has an allowlist.
4. **cline** — no sandbox; per-tool approval. Has an "auto-approve" UI.
5. **continue (agent mode)**, **crush** — per-tool approval, no sandbox.
6. **aider, mentat** — diff preview, but `aider` will run shell commands
   if you enable `--auto-commits` and a `/run` plus the model asks for it.
7. **gptme** — runs shell as you. Pretend it's `bash` with a smarter prompt.
8. **smol-developer**, **sweep** — write directly. `sweep` writes to a
   branch and opens a PR; `smol-developer` writes to your cwd.

If you can't tolerate "the model just ran `rm`", pick from the top half.

## 6. Prompt-cache strategies

This is where the cost differences live.

- **Anthropic prompt caching** (5-min ephemeral cache, ~10x cheaper reads):
  - opencode, cline, crush, OpenHands, aider — all set `cache_control` on
    the system prompt + tools block. Cline additionally caches the last few
    user/assistant turns.
- **OpenAI automatic prefix cache** (no API hint needed, just keep prefixes
  stable):
  - codex, continue, smol-developer — keep the system prompt stable across
    turns; the API caches automatically above 1024 tokens.
- **No explicit caching strategy** — mentat, gptme, sweep. Fine for short
  sessions, expensive for long ones.

Practical impact: a 30-minute coding session with cline + Sonnet is roughly
**3–5x cheaper** than the same session with mentat + Sonnet, purely because
of cache hits on the system prompt and tools block.

## 7. Telemetry posture

- **Off by default, opt-in to send anything**: opencode, aider, cline, crush,
  continue, mentat, gptme, smol-developer.
- **Opt-in but prompts on first run**: codex, OpenHands, plandex, sweep.

No CLI in this list ships "on by default, hidden opt-out" telemetry. If that
ever changes, the matrix in the top-level README will call it out.

## 8. What's missing across the board

Things *none* of these CLIs do well yet:

- **Cross-session memory.** All twelve start each session cold. "Remember our
  decision from last week" is the user's job (via files, not the tool).
- **Multi-repo edits in one transaction.** Each tool assumes one repo.
- **Cost dashboards.** A few show per-turn token counts; none show
  cumulative spend per project per month.
- **Deterministic replay.** You cannot rerun yesterday's session and get the
  same edits, even with `temperature=0`.

These are the obvious frontiers for a 13th entry to differentiate on.
