# ai-shell

> Snapshot date: 2026-04. Upstream: <https://github.com/BuilderIO/ai-shell>

A Node-based CLI that converts a natural-language sentence into a
**shell command**, then offers `[E]xecute / [R]evise / [C]opy /
[Cancel]`. Pure single-shot translation, no agent loop, no project
state, no file editing.

It earns its slot as a deliberate near-neighbor to
[`shell-gpt`](../shell-gpt/), [`shell-genie`](../shell-genie/),
[`gorilla-cli`](../gorilla-cli/), and [`tlm`](../tlm/) — the catalog
already has four "natural language → shell command" entries, and
`ai-shell` is the most popular of the bunch (5k+ stars) with the
distinct UX choice of a *revise loop* baked into the prompt.

## 1. Install footprint

- `npm install -g @builder.io/ai-shell` — exposes an `ai` binary.
- Requires Node 14+; the install is ~12 MB into the global `node_modules`.
- Per-user config at `~/.ai-shell` (a YAML-shaped key=value file)
  written by `ai config set OPENAI_KEY=...`.
- No daemon, no cache, no project-local state. Each `ai "..."`
  invocation is a fresh OpenAI HTTP call.

## 2. License

MIT.

## 3. Models supported

- OpenAI built-in: `gpt-4o`, `gpt-4`, `gpt-3.5-turbo`, configurable
  via `ai config set MODEL=...`.
- Any OpenAI-compatible endpoint via `ai config set
  OPENAI_API_ENDPOINT=https://...` (Ollama, LiteLLM proxy, vLLM,
  LM Studio, OpenRouter, Groq, DeepSeek). The wire format is
  hard-coded to OpenAI chat-completions, so non-compatible providers
  (native Anthropic, Gemini AI Studio) require a translating proxy.
- No native multi-provider switching like [`mods`](../mods/) or
  [`aichat`](../aichat/) — pick one endpoint per config.

## 4. MCP support

**No.** Single-shot translator; no tool calls, no MCP client.

## 5. Sub-agent model

**None.** One prompt → one shell-command suggestion → one user
decision. There is no agent loop and no chat session.

## 6. Telemetry stance

**Off.** No analytics. Network egress is exclusively to the OpenAI
endpoint configured. Nothing is logged to disk by default beyond the
config file.

## 7. Prompt-cache strategy

None. Each invocation sends the same short system prompt + user
sentence; OpenAI's automatic prefix caching may help on the system
block, but the tool does nothing to optimize for it.

## 8. Hot keybinds

The interactive prompt after generation is:

- `e` / Enter — Execute the suggested command.
- `r` — Revise: type a follow-up nudge ("use `find` not `fd`",
  "make it recursive") and the model regenerates.
- `c` — Copy to clipboard, do not execute.
- `Ctrl-C` / Esc — Cancel.

The `r` revise loop is what makes `ai-shell` distinct from its
neighbors — most one-shot shell-command CLIs only offer
execute / cancel.

## 9. Killer feature, weakness, when to choose

**Killer feature.** The **revise loop**. After the model proposes a
command, you can iteratively narrow it without retyping the original
intent: `ai "find large log files"` → suggests `find . -name "*.log"
-size +10M` → press `r` → "exclude node_modules" → regenerates →
press `e`. This is closer to a chat REPL than the `[E]xecute /
[D]escribe / [A]bort` triple in [`shell-gpt`](../shell-gpt/) or the
arrow-key candidate picker in [`gorilla-cli`](../gorilla-cli/),
and lower-friction than starting a full session. Bonus: `ai chat`
opens a multi-turn REPL when you actually want one.

**Weakness.** OpenAI-shape only. Hardcoded to chat-completions wire
format means native Anthropic / Gemini users need a proxy
(LiteLLM). No shell-aware prompt templating like
[`shell-genie`](../shell-genie/) (which ships bash / zsh / fish /
PowerShell variants); `ai-shell` assumes a POSIX-ish target.
Telemetry is off but the OpenAI endpoint sees every prompt — for
zero-network use, [`tlm`](../tlm/) (Ollama-only Go binary) is a
better pick.

**When to choose.**

- You want one `npm install -g` and an `ai "..."` that *also* lets
  you iterate on the suggestion without retyping the intent.
- You are already on OpenAI or have a LiteLLM-shaped local endpoint.
- You want a small `ai chat` fallback for the occasional multi-turn
  question without installing a second tool.

**When NOT to choose.**

- You want zero-cloud / fully local — pick [`tlm`](../tlm/) (Ollama
  + Go binary) or [`shell-genie`](../shell-genie/) with an Ollama
  endpoint.
- You want zero API key — pick [`tgpt`](../tgpt/) (free hosted
  providers) or [`gorilla-cli`](../gorilla-cli/) (free hosted Gorilla
  endpoint).
- You want shell-aware prompt templates (bash vs fish vs PowerShell)
  — pick [`shell-genie`](../shell-genie/).
- You want multi-candidate selection (N suggestions, arrow-key pick)
  — pick [`gorilla-cli`](../gorilla-cli/).
- You want a Unix pipe primitive rather than an interactive prompt
  — pick [`mods`](../mods/), [`smartcat`](../smartcat/), or
  [`llm`](../llm/).

## Representative invocation

```bash
$ npm install -g @builder.io/ai-shell
$ ai config set OPENAI_KEY=sk-...

$ ai "find files larger than 10MB but skip node_modules"

The generated script
find . -type f -size +10M -not -path "./node_modules/*"

> Run this script  [E]
> Revise query     [R]
> Copy to clipboard [C]
> Cancel           [X]

# press R, type "also exclude .git", press enter
The generated script
find . -type f -size +10M -not -path "./node_modules/*" -not -path "./.git/*"
```
