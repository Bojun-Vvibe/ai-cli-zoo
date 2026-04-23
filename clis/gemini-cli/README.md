# gemini-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/google-gemini/gemini-cli>

Google's official open-source terminal agent for the Gemini family. Node.js
TUI, generous free tier via personal Google account login, ReAct-style
agent loop with built-in tools (file ops, shell, web fetch, web search,
memory).

## 1. Install footprint

- `npm i -g @google/gemini-cli` (Node 20+) or `brew install gemini-cli`.
- Pure JS bundle, ~40 MB on disk after npm install (most of it is
  dependencies; the CLI itself is small).
- Runs on macOS, Linux, Windows. State in `~/.gemini/`.
- No daemon. Each invocation is a foreground TUI session or a one-shot
  `gemini -p "..."` call.

## 2. License

Apache-2.0.

## 3. Models supported

Gemini-only by design:

- Gemini 2.x Pro / Flash / Flash-Lite
- Gemini 1.5 Pro / Flash (legacy)
- Experimental thinking variants when Google exposes them

Auth paths:

1. **Personal Google login** (OAuth) — gives a free quota tied to your
   Google account. The default for new installs.
2. **AI Studio API key** (`GEMINI_API_KEY`) — pay-as-you-go.
3. **Vertex AI** (`GOOGLE_GENAI_USE_VERTEXAI=true` + ADC) — for GCP
   workloads, billed against a project.

There is no first-class adapter for non-Gemini providers. Community
forks exist but the upstream stance is "this is the Gemini CLI."

## 4. MCP support

**Yes — client only.** Configure under `mcpServers` in `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

Stdio and HTTP transports both work. Tools from MCP servers appear in the
same `/tools` list as built-ins, and Gemini routes function calls to them
transparently.

## 5. Sub-agent model

Single agent. The model emits parallel function calls when it wants to
fan out (e.g. read 5 files at once); the CLI executes them concurrently
with a small in-process pool. No spawnable child agents, no planner /
executor split. The "extended thinking" budget on Gemini 2.x Pro is the
closest thing to multi-step reasoning, and it's a model feature, not a
CLI feature.

## 6. Telemetry stance

**Opt-in but nudged.** First run prompts you. When enabled, sends usage
statistics (command names, error types, token counts — no prompt content)
to Google. Toggle with `gemini --telemetry off` or
`"telemetry": { "enabled": false }` in `settings.json`.

A separate **conversation logging** flag controls whether your prompts get
used for model improvement; this is OFF by default for paid API and Vertex,
and ON by default for the free personal-Google tier. Worth checking before
you paste anything sensitive.

## 7. Prompt-cache strategy

Uses Gemini's **context caching** API for sessions over the
provider-defined minimum (currently ~32k tokens). The CLI:

1. Builds a stable system prompt + tools schema prefix.
2. Once the conversation crosses the threshold, calls
   `cachedContents.create` and reuses the cache handle on subsequent turns.
3. Invalidates the cache when tools list changes or the user runs `/clear`.

You can see cache usage in the status bar (`cached: N tok`) and force a
refresh with `/cache clear`. Cache TTL defaults to 5 minutes — short, but
the context window is so large (1M+ on Pro) that re-priming is rarely a
practical problem.

## 8. Hot keybinds (TUI)

| Key | Action |
|-----|--------|
| `Esc` | Cancel current model call |
| `Ctrl+C` | Exit |
| `Ctrl+L` | Clear screen (keeps history) |
| `/clear` | Reset conversation |
| `/compress` | Summarize history into a shorter prefix |
| `/tools` | List available tools (built-in + MCP) |
| `/memory show` | Inspect the memory tool's saved facts |
| `/auth` | Switch between OAuth / API key / Vertex |
| `@<path>` | Inline-attach a file or directory to the next prompt |
| `!<cmd>` | Run a shell command directly (bypasses the model) |
| `Up Arrow` | Recall previous prompt |

## 9. Killer feature, weakness, when to choose

- **Killer:** **the 1M+ token context plus aggressive caching.** You can
  drop an entire mid-sized repo into one prompt with `@.` and keep
  iterating cheaply because the prefix is cached. For "explain this
  unfamiliar codebase" or "find every callsite of pattern X across 4000
  files" tasks, no other CLI in this catalog gives you that headroom out
  of the box.
- **Weakness:** **single-vendor lock-in plus a free-tier trust tradeoff.**
  If Gemini is wrong for your task, you can't swap models without
  abandoning the tool. And the default free tier opts your data into
  training — fine for OSS exploration, dangerous for proprietary code.
- **Choose it when:** you're doing exploratory work on a large unfamiliar
  codebase, you're already on Google Cloud / Vertex, or you want a free
  capable agent for personal projects and you're comfortable with the
  data terms.

## Pitfalls observed in real use

1. **Free-tier data policy.** Free OAuth quota = your prompts may be used
   for model training. Switch to API key or Vertex auth for any code
   you wouldn't paste into a public gist.
2. **`@` attachment expansion can blow up.** `@.` on a node project
   greedily includes `node_modules` unless you have a `.geminiignore`
   (same syntax as `.gitignore`). Always check ignore rules before
   attaching a directory.
3. **Cache invalidation on tool changes.** Hot-loading an MCP server
   mid-session blows the prompt cache; batch your tool config changes
   at session start.
4. **Shell tool on Windows** runs through `cmd.exe` not PowerShell by
   default; quoting bugs in long commands surface as cryptic
   `ENOENT` errors. Set `"shell": "pwsh"` in settings.json to switch.
5. **`/compress` is lossy** in predictable ways: it keeps the most recent
   exchanges verbatim and summarizes the older ones. If a critical fact
   sits 30 turns back, summarize manually with a pinned message instead
   of relying on `/compress`.
