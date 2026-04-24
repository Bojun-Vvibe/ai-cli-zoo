# ollama

> Snapshot date: 2026-04. Upstream: <https://github.com/ollama/ollama>
> Last verified version: **v0.21.2** (2026-04-23). License file:
> [`LICENSE`](https://github.com/ollama/ollama/blob/main/LICENSE) — MIT.

The local-LLM **runtime** that most other entries in this catalog
quietly delegate to. Ollama is not an agent and not a coding CLI in
the usual sense — it's a daemon plus a `pull` / `run` / `serve` /
`ps` command surface that fronts a `llama.cpp`-family backend with an
OpenAI-compatible HTTP endpoint.

It earns its own catalog slot because the *user interface* is a
genuine CLI (`ollama run llama3.1` drops you into a terminal chat
REPL, complete with multi-line input, `/?` help, `/save`, `/load`,
`/show`), and because ~half of the "Models supported" cells in the
matrix collapse to "whatever `ollama list` returns" once Ollama is on
the host. Knowing what Ollama itself does — and where it stops —
makes every other local-mode entry easier to reason about.

## 1. Install footprint

- macOS: `.dmg` or `brew install ollama`. Linux:
  `curl -fsSL https://ollama.com/install.sh | sh`. Windows: `.exe`
  installer. Container: `docker run ollama/ollama` (CPU) or
  `--gpus=all` (NVIDIA).
- Single Go binary plus a long-lived background service
  (`ollama serve`). On macOS the GUI app spawns the service; on Linux
  it ships as a `systemd` unit; on Windows it lives as a tray app.
- Models live under `~/.ollama/models/` (Linux/macOS) or
  `%USERPROFILE%\.ollama\models\` (Windows). A 7B Q4 model is ~4 GB,
  a 70B Q4 is ~40 GB — plan disk accordingly.
- Default HTTP endpoint: `http://127.0.0.1:11434`. Override with
  `OLLAMA_HOST=0.0.0.0:11434` to expose to the LAN (no auth — be
  careful).

## 2. License

MIT. License file at the repo root: `LICENSE`. The
underlying `llama.cpp` is also MIT. Individual *model weights* you
pull have their own licenses (Llama community license, Gemma terms,
Apache-2.0 for Qwen / DeepSeek-coder, etc.) — Ollama itself does
not relicense the weights and prints the model card on `ollama show`.

## 3. Models supported

Anything in the [Ollama library](https://ollama.com/library) plus
arbitrary GGUF files you import via a `Modelfile`. Highlights as of
v0.21:

- Llama 3.1 / 3.2 / 3.3 (1B – 405B)
- Qwen 2.5, Qwen 2.5-Coder (0.5B – 32B)
- Gemma 2 / 3
- DeepSeek-V3, DeepSeek-Coder, DeepSeek-R1 distills
- Phi-4, Phi-3.5
- Mistral, Mixtral, Nemotron, Granite-Code
- Embedding models: `nomic-embed-text`, `mxbai-embed-large`,
  `snowflake-arctic-embed`
- Vision: `llava`, `llama3.2-vision`, `qwen2.5-vl`, `minicpm-v`
- Custom: `ollama create mymodel -f Modelfile` to import any
  GGUF weight + system prompt + `PARAMETER` overrides

No closed-weight cloud models — by design.

## 4. MCP support

**No.** Ollama doesn't speak MCP itself. But because it exposes both
its native API and an **OpenAI-compatible** API at
`/v1/chat/completions`, any MCP-aware client in this catalog
(`opencode`, `codex`, `claude-code`, `cline`, `crush`,
`continue`, `gemini-cli`, `goose`) can target Ollama as the
*model* and bring its own MCP servers.

## 5. Sub-agent model

**None.** Ollama is a single-process serving layer. There is no
agent loop, no tool calls, no planner/builder split — those belong
to the client. Tool-call *protocol* is supported (`/api/chat` accepts
`tools: [...]` and the model emits structured calls when the model
itself was trained for it), but Ollama doesn't *execute* the tools.

## 6. Telemetry stance

**Off.** No phone-home, no usage analytics. Outbound network
traffic is exactly:

1. Pulls from the Ollama registry (`registry.ollama.ai`) when you
   `ollama pull` or `ollama run` a not-yet-cached model.
2. The model API endpoint you point clients at (your own
   `127.0.0.1:11434` by default).

Update checks happen on the macOS / Windows GUI app, not the CLI
binary. `OLLAMA_NOPRUNE=1` and `OLLAMA_KEEP_ALIVE=...` give you
control over local cache behavior.

## 7. Prompt-cache strategy

**KV-cache reuse across calls within a loaded-model lifetime.**
Ollama keeps a model loaded in VRAM/RAM for `OLLAMA_KEEP_ALIVE`
(default 5 minutes after last call) and reuses the KV cache for
prefix matches across calls. There is no provider-style "cache
control" header; it's automatic.

`OLLAMA_NUM_PARALLEL` (default 4 in v0.21) controls how many
concurrent requests share one loaded model. `OLLAMA_MAX_LOADED_MODELS`
caps how many distinct models can sit resident at once.

## 8. Hot keybinds (REPL + subcommands)

`ollama run <model>` is a chat REPL. Inside it:

| Command | Action |
|---------|--------|
| `/?` | Help |
| `/set system "..."` | Override the system prompt for this session |
| `/set parameter temperature 0.2` | Tune sampler at runtime |
| `/show info` | Print model card, parameters, license |
| `/show modelfile` | Print the Modelfile that built this model |
| `/save <name>` | Snapshot the current session as a new model |
| `/load <name>` | Resume a saved session |
| `/clear` | Reset conversation context |
| `/bye` or Ctrl-D | Exit |
| `"""` … `"""` | Multi-line input |

Top-level subcommands worth knowing:

| Subcommand | Action |
|------------|--------|
| `ollama list` | List local models with size / modified |
| `ollama ps` | List currently loaded models, GPU/CPU split, TTL |
| `ollama show <model>` | Modelfile, parameters, license, template |
| `ollama pull <model>` | Download or update |
| `ollama rm <model>` | Free disk |
| `ollama cp src dst` | Clone for a Modelfile-based override |
| `ollama create <name> -f Modelfile` | Build a custom model |
| `ollama serve` | Run the daemon explicitly (vs the GUI app) |
| `ollama run <model> "prompt"` | One-shot, stdout, no REPL |

## 9. Killer feature, weakness, when to choose

- **Killer:** **the path of least resistance to "I have a local LLM
  on this laptop."** `brew install ollama && ollama run llama3.1`
  works in under five minutes, on a Mac, with no Python env and no
  GPU configuration. The OpenAI-compatible endpoint then makes Ollama
  the *substrate* the rest of the catalog plugs into when its
  "Models supported" cell says "any OpenAI-compatible".
- **Weakness:** **opaque about what runs underneath.** It's
  `llama.cpp` with Ollama-flavored quantization defaults and a
  Modelfile abstraction on top. When a model is slow or low-quality,
  the levers (`num_ctx`, `num_gpu`, `num_thread`, `f16_kv`,
  quantization tier) are partially exposed but partially hidden
  behind defaults baked into each model's published Modelfile. Power
  users frequently end up `ollama show --modelfile`, copying it,
  editing `PARAMETER` lines, `ollama create`-ing a private variant.
  Also: no auth on the HTTP endpoint, ever — if you bind to anything
  other than `127.0.0.1`, put a reverse proxy in front.
- **Choose it when:** any other catalog entry's local mode says
  "Ollama" or "any OpenAI-compatible endpoint" and you don't already
  have one running. Pick [`ramalama`](../ramalama/) instead if you
  want **container-native, hardware-aware image selection**
  (NVIDIA / ROCm / Metal / oneAPI / Vulkan all addressed by the same
  CLI) and you'd rather pull a `quay.io/ramalama/...` image than
  manage an Ollama daemon. Pick neither — go to `llama.cpp`
  directly — if you need to control sampler internals, draft-model
  speculative decoding details, or grammar-constrained decoding at a
  level Ollama hides.

## Pitfalls observed in real use

1. **`OLLAMA_HOST=0.0.0.0:11434` is unauthenticated.** Anyone on the
   LAN can run inference on your GPU and read any session you save.
   Never expose 11434 directly; front it with a reverse proxy that
   adds auth, or keep it on `127.0.0.1` and tunnel over SSH.
2. **Default context is small.** Many models ship with `num_ctx`
   set to 4096 in their published Modelfile, even when the underlying
   weights support 128k+. Long-context use needs an explicit
   `/set parameter num_ctx 32768` (or a custom Modelfile) — and more
   VRAM than the default would have used. Check `ollama show
   --parameters <model>` before assuming a model is "broken at long
   inputs".
3. **`KEEP_ALIVE` interacts badly with VRAM-tight setups.** The
   default 5-minute keep-alive means a 30 GB model stays resident
   long after you Ctrl-C the REPL. On a 32 GB Mac, the next app
   you open will be paged. Set `OLLAMA_KEEP_ALIVE=0` to unload on
   every call, or `30s` for a sane middle ground.
4. **OpenAI compatibility is partial.** The
   `/v1/chat/completions` endpoint covers the common cases but not
   every field — `logprobs`, `tool_choice: required`, parallel
   tool-call indexing, and some streaming edge cases differ from
   OpenAI's spec. When a client misbehaves against Ollama but works
   against OpenAI, suspect the compatibility layer first.
5. **`ollama pull` is not atomic across the registry-mirror caches.**
   On a flaky connection, a partial blob can land in
   `~/.ollama/models/blobs/` and a subsequent run will report
   "checksum mismatch" or, worse, hang at startup. Fix:
   `ollama rm <model> && ollama pull <model>`. Don't try to
   manually delete blob files — the manifest will go stale.
6. **The macOS menu-bar app and `ollama serve` collide.** If you
   `brew install ollama` *and* run the `.app`, you'll have two
   daemons competing for port 11434 and the second will silently
   fail. Pick one: either run the GUI (recommended on macOS) or
   `brew services start ollama` and quit the app.
