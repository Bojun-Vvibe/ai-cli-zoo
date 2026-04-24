# qwen-code

> Snapshot date: 2026-04. Upstream: <https://github.com/QwenLM/qwen-code>

Alibaba's official open-source terminal agent for the **Qwen3-Coder**
family. A `gemini-cli` fork at the architectural level — same Node TUI
shape, same ReAct-style loop, same MCP client surface — retuned for
Qwen's tool-call format and shipped with a generous free tier through
Alibaba Cloud's Model Studio.

This entry exists to cover the "what if I want a serious coder model
that isn't Anthropic, OpenAI, or Google?" question. Qwen3-Coder
benchmarks at frontier-adjacent levels on SWE-Bench-Verified at a
fraction of the price, and `qwen-code` is the only first-party CLI
that targets it directly.

## 1. Install footprint

- `npm i -g @qwen-code/qwen-code` (Node 20+) or run via
  `npx @qwen-code/qwen-code`. macOS / Linux / Windows.
- ~50 MB on disk after install (transitively pulls in the same TUI
  and tool stack as upstream `gemini-cli`).
- Per-user state at `~/.qwen/` (config, sessions, OAuth tokens).
- No daemon. Foreground TUI session, or one-shot
  `qwen -p "fix the failing test"` for scripting.

## 2. License

Apache-2.0.

## 3. Models supported

Qwen-first, but the OpenAI-compatible adapter means almost any
provider works in practice:

- **Qwen3-Coder** (480B-A35B Instruct) — the headline model, available
  via Alibaba Cloud Model Studio with an OAuth-tied free quota.
- **Qwen3-Coder Plus / Flash** — the hosted tiers behind the same API.
- Any **OpenAI-compatible endpoint** via `OPENAI_API_KEY` +
  `OPENAI_BASE_URL` overrides — covers OpenRouter, Together, vLLM,
  llama.cpp server, Ollama, and friends.
- DashScope endpoints (Alibaba's pay-as-you-go API) for production
  use beyond the free tier.

There is no first-class adapter for Anthropic or Gemini — those
work only if you front them with an OpenAI-compatible proxy.

Auth paths:

1. **Qwen OAuth** — `qwen` triggers a browser flow tied to your
   Alibaba Cloud account, gives you a free daily quota on
   Qwen3-Coder. The default for new installs.
2. **DashScope API key** (`DASHSCOPE_API_KEY`) — pay-as-you-go.
3. **Generic OpenAI-compatible** (`OPENAI_API_KEY` +
   `OPENAI_BASE_URL`) — point at anything you want.

## 4. MCP support

**Yes — client only.** Inherited from the `gemini-cli` lineage.
Configure under `mcpServers` in `~/.qwen/settings.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

Tool schemas are merged at session start. There is no MCP _server_
mode — `qwen-code` does not expose its internals over MCP.

## 5. Sub-agent model

**Single-agent with parallel tool calls.** No `Task`-style sub-agent
spawn. The loop is:

1. User message in.
2. Model emits zero-or-more tool calls (potentially in parallel).
3. Tool results are fed back.
4. Repeat until the model returns a plain text response.

If you want sub-agents on Qwen3-Coder, run `qwen-code` underneath
`opencode` or `forge` as the model backend instead.

## 6. Telemetry stance

**Opt-in.** No usage telemetry ships without `telemetry.enabled: true`
in `~/.qwen/settings.json`. The OAuth path obviously talks to Alibaba
Cloud (that's how the free tier accounting works), but tool-call
contents and file diffs are not telemetered separately.

DashScope and other API-key paths send only the request payloads to
the configured endpoint — same posture as any other provider.

## 7. Prompt-cache strategy

Qwen3-Coder over DashScope supports **automatic prefix caching** —
no explicit cache hints needed; the system prompt and stable tool
schemas are deduplicated server-side after the first call. The CLI
exposes the resulting cache hit ratio in the status line.

For non-Qwen OpenAI-compatible endpoints, caching depends on the
provider. Local models (Ollama, vLLM) do not cache across calls
unless you front them with a KV-cache-aware server.

The CLI also implements **client-side context truncation** with a
sliding window once the context budget is exceeded — older turns are
dropped before older tool results, on the theory that fresh tool
results are more load-bearing than chat history.

## 8. Hot keybinds (TUI)

- `Ctrl-C` — interrupt the running tool call (does not kill session).
- `Ctrl-D` — exit, optionally saves transcript to `~/.qwen/sessions/`.
- `Esc` — drop the in-progress draft.
- `/help` — list commands.
- `/auth` — switch between OAuth and API-key paths.
- `/model <name>` — swap model mid-session (works across providers
  when both are configured).
- `/stats` — show token usage, cache hit ratio, dollar cost.
- `/memory` — view the on-disk memory file the agent reads at boot
  (`QWEN.md` in CWD, then `~/.qwen/QWEN.md`).
- `Shift-Tab` — toggle "auto-approve all tool calls" (sticky for the
  session).

## 9. Killer feature, weakness, when to choose

**Killer feature.** First-class Qwen3-Coder access with a non-trivial
free tier and a TUI that's already battle-tested (it's a fork of
`gemini-cli`'s TUI, which has years of polish behind it). The combo
of "frontier-adjacent coder model" + "no credit card needed for the
first quota" is unique in the zoo. If you want to evaluate whether
non-Western frontier coder models are good enough for your workflow,
this is the lowest-friction way to find out.

**Weakness.** The fork relationship with `gemini-cli` is a double-edged
sword: bug fixes and features land downstream from the upstream Google
project, with a delay measured in weeks. Some Gemini-specific code
paths still exist in the codebase and have to be maintained even
though they aren't exercised in the Qwen variant. There's also a
real risk that an upstream rebase will land before a downstream patch,
forcing the Qwen team to redo work. From a user POV: if a feature
appears in `gemini-cli` today, expect ~2-6 weeks for `qwen-code`.

**When to choose `qwen-code`.**

- You want to try **Qwen3-Coder** in an agentic setup without writing
  your own tool scaffolding. The CLI handles the tool-call format
  quirks for you.
- You want a **free-tier-first** experience. The OAuth flow gives
  you real daily quota; you can do meaningful work without a credit
  card on file.
- You want a **`gemini-cli`-shaped UX** but pointed at a different
  model family. Same keybinds, same MCP config shape, same
  one-shot-vs-TUI split.

**When not to.** If you want to use Qwen3-Coder _alongside_ Claude
or GPT in the same workflow, route Qwen as a backend through
`opencode`, `forge`, or `aider` instead — those handle multi-provider
sessions natively. If you want a polished single-binary TUI with no
Node dependency, this is not it; pick `crush` or `codex`. And if
free-tier quota is _not_ a constraint, the architectural advantages
over `gemini-cli` are thin enough that the choice comes down to
"which model do you actually want to use."
