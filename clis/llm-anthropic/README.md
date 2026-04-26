# llm-anthropic

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/llm-anthropic>
> Last verified version: **0.25** (released 2026-04-16). License file:
> [`LICENSE`](https://github.com/simonw/llm-anthropic/blob/main/LICENSE)
> (blob sha `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`) — Apache-2.0.

The first-party Anthropic provider plugin for [`llm`](../llm/). Adds the
Claude family (Opus, Sonnet, Haiku — including the latest 4.x line and
extended-thinking variants) as registered models inside the same `llm`
surface that hosts OpenAI, Gemini, Mistral, and friends. Once installed
you get `llm -m claude-sonnet-4 "..."`, `llm chat -m claude-opus-4 ...`,
streaming, attachments (PDFs / images), tool calling, prompt caching
markers, and conversation logging into the same SQLite database `llm`
already writes for every other provider.

## 1. Install footprint

- `llm install llm-anthropic` (uses `llm`'s plugin installer, which
  shells out to `pip` inside the `llm` venv).
- Pure Python, depends on the `anthropic` SDK (~5 MB).
- Requires `llm` >= 0.26 and Python 3.9+.
- Configuration: `llm keys set anthropic` (key stored at
  `~/Library/Application Support/io.datasette.llm/keys.json` on macOS,
  XDG-equivalent on Linux). No daemon, no extra config files.

## 2. License

Apache-2.0. License file at the repo root: `LICENSE` (blob sha
`261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64` at tag `0.25`). Same license
as the host `llm` framework, so the combined install is unambiguously
Apache-2.0.

## 3. Models supported

Just Anthropic — that is the entire point of the plugin. As of 0.25:

- `claude-opus-4-20250514`, `claude-opus-4-1`, alias `claude-opus-4`
- `claude-sonnet-4-20250514`, `claude-sonnet-4-5`, alias `claude-sonnet-4`
- `claude-3-7-sonnet-latest` (with optional `-thinking` suffix to enable
  extended thinking)
- `claude-3-5-sonnet-latest`, `claude-3-5-haiku-latest`
- Legacy `claude-3-opus`, `claude-3-haiku`

Vendor endpoints (Bedrock / Vertex) are not handled by this plugin —
use `llm-bedrock` / `llm-vertex` for those.

## 4. MCP support

**No.** `llm-anthropic` exposes Anthropic's *own* tool-use schema
(function-call style) so `llm`'s `--tool` / fragment-tool features work
against Claude. It does not speak MCP. If you want MCP servers wired
into Claude calls, use `claude-code` or `goose` — `llm`'s tool layer is
its own.

## 5. Sub-agent model

**None.** One `llm -m claude-...` call → one HTTP request. The plugin
is a translator, not an agent loop. Multi-turn happens via `llm chat`
or by passing `-c` (continue) to thread replies; both are still single
linear conversations.

## 6. Telemetry stance

**Off.** Inherits `llm`'s "no analytics, log everything to local SQLite"
posture. Every prompt + response goes to
`~/Library/Application Support/io.datasette.llm/logs.db` (queryable with
`llm logs`). The only network egress is `api.anthropic.com`.

## 7. Prompt-cache strategy

**Yes — and this is the headline feature in 0.2x.** The plugin emits
Anthropic `cache_control: {"type": "ephemeral"}` markers on:

- The system prompt (when `-o cache_system 1` or via fragment).
- Long attachments / tool results (auto-marked above a size threshold).
- The last assistant turn in `llm chat` so multi-turn replays hit the
  5-minute cache window.

This drops cost ~10× on long-system-prompt workflows (e.g., feeding a
big spec file into every call). Look at `cache_creation_input_tokens`
and `cache_read_input_tokens` in `llm logs -n 1 --json` to confirm.

## 8. Hot keybinds (REPL + subcommands)

There is no bespoke REPL — everything is `llm` subcommands with `-m
claude-...`:

| Invocation | Action |
|------------|--------|
| `llm -m claude-sonnet-4 "..."` | One-shot prompt |
| `llm chat -m claude-opus-4` | Multi-turn REPL |
| `llm -m claude-sonnet-4 -a screenshot.png "..."` | Attach an image |
| `llm -m claude-3-7-sonnet-latest-thinking "..."` | Force extended thinking |
| `llm -m claude-sonnet-4 -o max_tokens 8192 -o temperature 0.2 "..."` | Per-call options |
| `llm logs -m claude-sonnet-4 -n 5` | Replay recent calls from SQLite |

In `llm chat`, the REPL keybinds are `prompt_toolkit` defaults (Ctrl-D
to exit, Ctrl-C to interrupt streaming, `!multi` for multi-line input).

## 9. Killer feature, weakness, when to choose

- **Killer:** **first-party prompt-cache wiring without you having to
  hand-build cache_control blocks.** Most plugins for other frameworks
  (LangChain providers, raw `anthropic` SDK calls) make you place the
  markers manually. `llm-anthropic` does it from a single CLI flag and
  surfaces the cache hit/miss counters in the SQLite log — so you can
  actually see whether your workflow is benefiting.
- **Weakness:** **provider-locked.** This plugin is Claude only. If
  you want to A/B the same prompt across Claude and GPT-5 in the same
  command, you need `llm-anthropic` *and* `llm-openai-plugin` and your
  own shell loop. The `llm` framework supports that fine — but the
  plugin itself does not abstract.
- **Choose it when:** you already use [`llm`](../llm/) as your "every
  prompt logged to SQLite" substrate and you want Claude available in
  the same surface as the OpenAI built-in. Pick
  [`llm-mistral`](../llm-mistral/) for Mistral coverage,
  [`llm-gemini`](../llm-gemini/) for Gemini, or
  [`claude-code`](../claude-code/) if you actually want a coding agent
  loop with file edits + MCP rather than raw chat.

## Pitfalls observed in real use

1. **Model alias drift.** `claude-sonnet-4` is an alias that follows
   whichever Sonnet 4.x is current at upstream. If you need
   reproducible runs (eval suites, paper benchmarks), pin the dated
   form `claude-sonnet-4-20250514` instead — the alias *will* roll
   under you when 4.5 ships.
2. **Cache markers don't survive `llm logs --to ...` replay.** Replaying
   an old conversation through `llm` re-issues the call without the
   original cache_control hints. The replayed call is a cache miss and
   priced normally — fine for ad-hoc, surprising on a $-tracked eval
   harness.
3. **Extended thinking eats tokens silently.** `-thinking` variants
   emit a hidden chain-of-thought block that counts against output
   tokens but isn't shown in `llm` chat output by default. Use `llm -m
   ...-thinking -o thinking 1 ...` to surface it; otherwise your bill
   looks weird.
4. **Attachments larger than 5 MB get base64-bloated to >10 MB on the
   wire** because `llm`'s attachment layer doesn't stream uploads.
   Anthropic's HTTP timeout is generous but uploading a 200-page PDF
   from a hotel Wi-Fi will fail.
5. **`llm install llm-anthropic` lands the plugin in whichever Python
   env hosts `llm`.** If you have `llm` via `pipx` *and* a system `pip
   install llm`, only one of them gets the plugin. Check `which llm`
   before installing — and prefer `pipx inject llm llm-anthropic` on a
   pipx setup.
6. **Tool-use schema differs from OpenAI's.** If you've written
   `llm`-fragment tools assuming OpenAI's JSON schema dialect, Claude's
   stricter validation will reject some patterns (e.g., empty `enum`
   arrays, untyped `additionalProperties: true`). The plugin surfaces
   the upstream error verbatim, which is helpful but not auto-corrected.
