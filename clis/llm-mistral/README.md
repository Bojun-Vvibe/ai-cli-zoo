# llm-mistral

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/llm-mistral>
> Last verified version: **0.15** (released 2025-07-16). License file:
> [`LICENSE`](https://github.com/simonw/llm-mistral/blob/main/LICENSE)
> (blob sha `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`) — Apache-2.0.

The Mistral provider plugin for [`llm`](../llm/). Adds the entire
Mistral hosted line-up — Large, Medium, Small, Codestral, Pixtral,
Mistral Embed — as registered models inside `llm`'s standard surface.
Once installed: `llm -m mistral-large "..."`, `llm chat -m
codestral-latest`, streaming, function calling, embeddings, all logged
into the same SQLite database `llm` keeps for every other provider.

## 1. Install footprint

- `llm install llm-mistral` (uses `llm`'s plugin installer).
- Pure Python, depends on `httpx` (already a transitive dep of `llm`)
  and a thin async wrapper. No extra C-extensions.
- Requires `llm` >= 0.13 and Python 3.9+.
- Configuration: `llm keys set mistral` (key stored in `llm`'s standard
  keys.json). No daemon, no extra config files.

## 2. License

Apache-2.0. License file at the repo root: `LICENSE` (blob sha
`261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64` at tag `0.15`). Identical
text to the host `llm` framework, so a `llm + llm-mistral` install is
unambiguously Apache-2.0.

## 3. Models supported

Just Mistral's hosted endpoints (`api.mistral.ai`). As of 0.15 this
covers the full menu:

- **Chat:** `mistral-tiny`, `mistral-small`, `mistral-medium`,
  `mistral-large`, `mistral-large-latest`, `open-mistral-7b`,
  `open-mixtral-8x7b`, `open-mixtral-8x22b`, `ministral-3b-latest`,
  `ministral-8b-latest`.
- **Code-tuned:** `codestral-latest`, `codestral-mamba-latest`.
- **Vision:** `pixtral-12b-latest`, `pixtral-large-latest`.
- **Embeddings:** `mistral-embed`.

Self-hosted Mistral via vLLM / Ollama is *not* this plugin's job — for
those, point `llm-openai-plugin` at the OpenAI-compatible endpoint or
use `llm-ollama`.

## 4. MCP support

**No.** The plugin exposes Mistral's function-calling schema so `llm`'s
fragment-tools work, but it does not speak MCP. (Mistral's hosted API
does not expose MCP either; this is a faithful translation, not a gap.)

## 5. Sub-agent model

**None.** One `llm -m mistral-...` invocation → one HTTP call. Use
`llm chat -m mistral-large` for multi-turn; everything else is single
linear conversation.

## 6. Telemetry stance

**Off.** Inherits `llm`'s "no analytics, log everything to local SQLite"
posture. Egress is exclusively `api.mistral.ai`. Every call writes to
`logs.db` — `llm logs -m mistral-large -n 5 --json` is the inspection
loop.

## 7. Prompt-cache strategy

**None at the plugin level.** Mistral's API does not yet expose a
prompt-cache primitive (no `cache_control` analogue), so the plugin has
nothing to wire up. Long-system-prompt workflows on Mistral will pay
full input-token cost on every call — which is the main reason to
prefer `llm-anthropic` for that pattern.

The plugin *does* re-use `httpx` connection pooling so back-to-back
calls in the same Python process avoid TLS handshake overhead, but
that's connection-level, not token-level.

## 8. Hot keybinds (REPL + subcommands)

There is no bespoke REPL — everything runs through `llm` with
`-m mistral-...`:

| Invocation | Action |
|------------|--------|
| `llm -m mistral-large "..."` | One-shot prompt |
| `llm -m codestral-latest "explain this code" < file.py` | Pipe code in |
| `llm chat -m mistral-large` | Multi-turn REPL |
| `llm -m pixtral-large-latest -a screenshot.png "..."` | Vision call |
| `llm embed-multi docs -m mistral-embed --files docs '*.md'` | Bulk embed corpus |
| `llm -m mistral-large -o temperature 0.1 -o random_seed 7 "..."` | Pin sampling for repro |
| `llm logs -m codestral-latest -n 10` | Replay recent code calls |

REPL keybinds in `llm chat` are `prompt_toolkit` defaults: Ctrl-D to
exit, Ctrl-C to interrupt streaming, `!multi` for multi-line input.

## 9. Killer feature, weakness, when to choose

- **Killer:** **Codestral access in the same Unix-pipe-friendly surface
  as every other provider.** `git diff | llm -m codestral-latest
  "summarize this change"` works the same way `git diff | llm -m gpt-4o
  "..."` does. No SDK, no asyncio, no auth dance per call — `llm keys
  set mistral` once and it's a pipe target for the rest of your shell
  life. The embeddings command (`llm embed-multi` against
  `mistral-embed`) is the cheapest decent-quality embeddings path
  available without running anything locally.
- **Weakness:** **no prompt cache means long-context workflows are
  expensive.** A 30 KB system prompt + 100 chat turns will pay full
  input cost every turn on Mistral, vs. 10× cheaper on Claude with
  `llm-anthropic`'s cache markers. Also: vision (Pixtral) is supported
  but the upload path doesn't stream, so >5 MB images get the same
  bloated-base64 problem as `llm-anthropic`.
- **Choose it when:** you already use [`llm`](../llm/) and want
  Mistral's open-weights pedigree and EU-data-residency story alongside
  your other providers; or you specifically want Codestral as a code
  model in a one-shot pipe. Pick [`llm-anthropic`](../llm-anthropic/)
  for Claude (and prompt caching), [`llm-gemini`](../llm-gemini/) for
  Gemini, or wire `llm-openai-plugin` at a self-hosted Mistral
  (`vllm`-served) endpoint if you want to keep weights on-prem.

## Pitfalls observed in real use

1. **`mistral-large` aliases drift.** `mistral-large-latest` follows
   upstream re-releases. For eval reproducibility, pin the dated form
   (e.g., `mistral-large-2407`) — discoverable via `llm models`.
   `mistral-large` (no suffix) maps to whatever `mistral-large-latest`
   currently resolves to and *will* roll.
2. **Function-calling schema is stricter than OpenAI's.** Mistral
   rejects tool definitions that omit `parameters` even when the tool
   takes none — pass `parameters: {"type":"object","properties":{}}`
   explicitly. The plugin forwards the upstream 400 verbatim, which is
   helpful but not auto-corrected.
3. **`mistral-embed` returns 1024-dim vectors only.** No
   `dimensions=512` truncation like OpenAI's `text-embedding-3-*`. If
   your vector store is sized for 1536 or 768 you have to migrate.
4. **Streaming sometimes drops the final chunk on flaky connections**
   and `llm` records the partial response in SQLite as if it were
   complete. Re-run with `--no-stream` for any call whose output you
   plan to act on programmatically.
5. **`llm install llm-mistral` lands in whichever Python env hosts
   `llm`.** Same pipx-vs-system gotcha as every other `llm` plugin —
   check `which llm` first, then prefer `pipx inject llm llm-mistral`
   on a pipx setup.
6. **Pixtral attachments need explicit MIME hinting.** `llm -a
   image.heic ...` will be sent with the wrong content-type and
   Mistral will 415. Convert HEIC → JPEG/PNG first; the plugin doesn't
   transcode.
7. **No native JSON-mode flag.** Other providers expose
   `response_format: {"type": "json_object"}`; Mistral's equivalent is
   the `response_format` API field but the plugin's `-o` options don't
   surface it as a typed flag in 0.15. Workaround: pass via `--option
   response_format '{"type":"json_object"}'` and verify the model
   actually honored it (older Mistral models silently ignore).
