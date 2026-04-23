# goose

> Snapshot date: 2026-04. Upstream: <https://github.com/block/goose>

Block's open-source on-machine AI agent. Rust core, Python/JS extensions,
MCP-native from day one. Optimized for "do the whole task locally, then show
me the diff."

## 1. Install footprint

- Single binary via `brew install block-goose-cli` or curl-installer
  (`download.goose.ai/install.sh`). ~30 MB.
- Optional Electron desktop app (`Goose Desktop`) wraps the same core.
- Runs on macOS, Linux, Windows. State in `~/.config/goose/` (XDG-compliant).
- No background daemon; each `goose session` is a foreground process.

## 2. License

Apache-2.0.

## 3. Models supported

Provider-agnostic. First-class drivers for:

- Anthropic (Claude family)
- OpenAI (GPT-4.x / o-series / GPT-5)
- Google (Gemini 1.5/2.x via Vertex or AI Studio)
- Groq, Together, OpenRouter
- Local: Ollama, llama.cpp HTTP, vLLM, LM Studio
- Databricks, Bedrock, Azure OpenAI

Provider switch lives in `~/.config/goose/config.yaml` under `GOOSE_PROVIDER`.
You can set per-recipe overrides — handy when a heavy planner uses Claude
Opus and the implementer uses a local 32B.

## 4. MCP support

**Yes — client + server, and arguably the most aggressive MCP user in this
catalog.** Every built-in tool (developer, computer-controller, memory,
jetbrains, tutorial, etc.) is implemented as an internal MCP "extension."
Adding a third-party server is a one-liner:

```bash
goose session --with-extension "npx -y @modelcontextprotocol/server-github"
```

Or persist it in `config.yaml` under `extensions:`. Stdio, SSE, and
streamable-HTTP transports are all supported.

## 5. Sub-agent model

No dynamic sub-agent spawning. Instead, goose exposes **recipes** — YAML
files that pin a (model, prompt, extensions, parameters) bundle. You can
invoke another recipe as a subroutine via the `subagent` extension, which
opens a child goose process and returns its final message. This is closer to
Unix-style composition than to a parent/child agent tree.

## 6. Telemetry stance

**Opt-in.** Disabled by default. `GOOSE_TELEMETRY_ENABLED=true` flips it on.
When enabled, sends anonymized event names (`session_start`, `tool_call`,
errors) — no prompt content. The list of events is documented in `docs/`
and the code path is auditable in `crates/goose/src/tracing/`.

## 7. Prompt-cache strategy

Per-provider:

- Anthropic: emits `cache_control: ephemeral` on the system prompt and on
  the tools schema. Conversation prefix is also cached when length crosses
  the provider's minimum-token threshold.
- OpenAI: relies on the API's automatic prefix cache; goose stabilizes the
  system prompt + tool order to maximize hits.
- Local providers: no cache markers; goose lets the runtime do its own
  KV reuse.

Cache hit rate is observable via `goose session --debug` (look for
`cache_creation_input_tokens` vs `cache_read_input_tokens` in the
provider response trace).

## 8. Hot keybinds (TUI / desktop)

| Key | Action |
|-----|--------|
| `Ctrl+C` once | Cancel current model call (keeps session) |
| `Ctrl+C` twice | Exit session |
| `/exit` | Save and quit |
| `/extension <name>` | Hot-load an MCP extension mid-session |
| `/recipe <path>` | Switch to a different recipe without restarting |
| `/builtin developer` | Re-enable shell + file tools if you started lean |
| `/?` | Help |
| `Up Arrow` | Recall previous input |

In Goose Desktop, the same slash commands work in the chat box; native
keybinds are macOS-standard (`Cmd+K` clears, `Cmd+,` settings).

## 9. Killer feature, weakness, when to choose

- **Killer:** **everything-is-an-MCP-extension.** Because the agent's own
  built-in tools live behind the same interface as third-party servers, you
  can mix and match without recompiling. Want a session with only the
  `memory` extension and a custom Postgres MCP? One flag. Want to ship a
  reproducible "code reviewer" persona to your team? Check in a recipe
  YAML — they get the same agent.
- **Weakness:** **discovery cost.** The flexibility means there's no single
  "goose way" to do a task; new users hit decision fatigue (which provider?
  which extensions? recipe or session?). Documentation has improved a lot
  but the surface area is real.
- **Choose it when:** you're already invested in MCP, you want a CLI that
  treats local and remote tools symmetrically, and you'd rather express
  workflows as composable YAML than as imperative scripts. Especially
  strong if your shop runs a heterogeneous model fleet (some Anthropic,
  some local) and you want one client across all of them.

## Pitfalls observed in real use

1. **Recipe drift.** Recipes pin model names but not model versions; a
   provider deprecation breaks every recipe silently. Pin via aliases in
   your provider config, not in the recipe.
2. **Extension order matters.** When two extensions expose tools with the
   same name, the *last loaded wins.* Surprising if you `--with-extension`
   on the CLI on top of a `config.yaml` extension list.
3. **Computer-controller on macOS** needs Accessibility + Screen Recording
   permissions; first run fails silently if you skip the prompt. Re-grant
   under System Settings → Privacy & Security.
4. **Subagent recipe** spawns a fresh process per call — token cost adds up
   fast for tight loops. Prefer it for "go do this 5-minute task" style
   delegation, not for "score each of these 200 candidates."
