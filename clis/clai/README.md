# clai

> Snapshot date: 2026-04. Upstream: <https://github.com/baalimago/clai>

"**Command-Line Artificial Intelligence**" — a single Go binary that
acts as a Unix-pipe-shaped LLM client with a built-in
**context-feeder** for local files, URLs, and tool output. Sits in
the same neighbourhood as `mods` / `aichat` / `smartcat` (one-shot
stdin→stdout LLM filter) but leans harder into "feed it a glob of
files plus a question and pipe the answer onward", with text-to-image
and text-to-speech modes bolted on as separate sub-commands.

## 1. Install footprint

- `go install github.com/baalimago/clai@latest`, or grab a release
  binary from <https://github.com/baalimago/clai/releases>.
- Single static Go binary, no runtime deps.
- Config at `~/.config/clai/` (XDG-respecting): `textConfig.json`,
  `photoConfig.json`, `ttsConfig.json`, plus per-vendor `*.json`
  files containing API endpoints and model defaults. Conversations
  saved to `~/.config/clai/conversations/`.

## 2. Repo + version + license

- Repo: <https://github.com/baalimago/clai>
- Latest release: **v1.10.6**
- License: **MIT** —
  <https://github.com/baalimago/clai/blob/main/LICENSE>
- Default branch: `main`

## 3. Models supported

OpenAI, Anthropic, Mistral, **Novita**, **DeepSeek**, Ollama (local),
plus any OpenAI-compatible endpoint via the `openai` vendor block.
Switch with `clai -cm <model>` per call or set `model:` in
`textConfig.json`.

## 4. MCP support

None. clai's "tools" are the shell glob you pipe in, the URLs it
fetches when you pass `-i URL`, and the conversation file it can
re-load. No tool-call protocol.

## 5. Sub-agent model

None. Single conversation, single model per invocation. You get
parallelism by pipelining shell processes, not by spawning sub-agents.

## 6. Telemetry stance

Off, no opt-in. Egress is exclusively to the vendor endpoints you
configured.

## 7. Killer feature, weakness, when to choose

**Killer feature.** First-class **glob-as-context** flag:
`clai -i 'src/**/*.go' query "audit error handling"` packs every
matched file into the prompt automatically (with file headers the
model can name back) — no `cat … | clai` dance, no separate packer
step. Combined with `-i URL` for inline web fetch and conversation
re-entry via `clai r <name> "follow-up"`, it covers the same
day-to-day "ask the model about my files / a URL / last week's
chat" surface as a much heavier TUI for one binary's worth of
install.

**Weakness.** Provider matrix is shallower than `aichat` (no Gemini
/ Cohere / Bedrock / Vertex / Qwen first-class). Image / TTS
sub-commands feel grafted on; if image generation is the goal you
want a real image CLI. Conversation files are JSON blobs, not
Markdown — less `git diff`-friendly than `chatblade`'s
plain-text history.

**When to choose.**

- You want **`mods`-style Unix-pipe ergonomics** but with a
  built-in glob/URL **context feeder** so you do not have to chain
  `files-to-prompt` / `repomix` first.
- You target the **OpenAI / Anthropic / DeepSeek / Ollama** lane
  and do not need the long tail of providers `aichat` covers.
- You want **named, re-enterable conversations** from a single Go
  binary without standing up `llm`'s plugin ecosystem.

**When not to choose.** You need MCP, you need 20+ provider
coverage including Chinese-ecosystem models (→ `aichat`), you want
a TUI (→ `oterm` / `elia` / `parllama`), or you want an editing
agent (→ `aider` / `opencode`).

## 8. Quick usage

```sh
go install github.com/baalimago/clai@latest

# one-shot, default vendor (openai)
clai query "explain the consensus algorithm in raft in three bullets"

# glob-as-context: pack all Go files in src/ into the prompt
clai -i 'src/**/*.go' query "audit error handling and suggest fixes"

# fetch a URL inline as context
clai -i https://example.com/post.html query "summarise the headline argument"

# pick a different model on this call
clai -cm claude-3-7-sonnet-latest query "..."

# re-enter a saved conversation
clai r raft-notes "what about leader election timeouts?"
```
