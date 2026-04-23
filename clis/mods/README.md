# mods

> Snapshot date: 2026-04. Upstream: <https://github.com/charmbracelet/mods>

Charm's minimalist LLM CLI. Not an agent — a **unix pipe primitive** for
LLMs. Reads stdin, asks a model, writes to stdout. No file editing,
no tool calls, no TUI session, no project state. Designed to compose
with the rest of your shell rather than replace it.

This entry exists in the catalog as a deliberate counterpoint: every
other tool here is a stateful agent. `mods` shows what the opposite
extreme looks like, and is genuinely the right answer for a surprising
share of "I just need an LLM to do this one thing" situations.

## 1. Install footprint

- `brew install charmbracelet/tap/mods`, `go install github.com/charmbracelet/mods@latest`,
  or grab a release binary from GitHub.
- Single Go binary, ~25 MB, no runtime dependencies.
- Per-user config at `~/.config/mods/mods.yml` (or
  `$XDG_CONFIG_HOME/mods/mods.yml`). Optional cache at
  `~/.cache/mods/`.
- No daemon, no project-local state, no log database. Each invocation
  is a fresh process unless you opt into the conversation cache.

## 2. License

MIT.

## 3. Models supported

Multi-vendor via per-API config blocks. Out of the box:

- OpenAI (GPT-4.x, o-series, omni)
- Anthropic (Claude family, all current generations)
- Google (Gemini Pro / Flash via AI Studio API)
- Cohere
- Groq
- Perplexity
- Local OpenAI-compatible endpoints (Ollama, LM Studio, llama.cpp
  server, vLLM, etc.)
- Azure OpenAI
- Any other OpenAI-compatible API by adding an `api:` block

You name "models" in `mods.yml` with arbitrary aliases (`gpt`, `claude`,
`local`, `fast`, `smart`) and switch between them with `-m <alias>`.
This is the cleanest provider-aliasing UX in the catalog — gptme
and aider both support multi-provider but make you specify the full
canonical model id every time.

## 4. MCP support

**No.** And by design — `mods` is one-shot and stateless, so the MCP
session-handshake model doesn't fit. If you need tool calls, use a
real agent CLI instead.

## 5. Sub-agent model

**No.** No agents at all. Stdin → model → stdout. Your "agent loop" is
whatever shell pipeline you compose around it.

## 6. Telemetry stance

**Off. No phone-home at all.** No usage stats, no error reporting, no
update check unless you opt in. The binary makes exactly one outbound
network connection per invocation: to the LLM API endpoint you
configured. This is the most privacy-preserving stance in the catalog.

## 7. Prompt-cache strategy

**Per-conversation cache (opt-in), no provider-side prefix caching.**
With `--continue` or `-C <name>`, mods writes the conversation to a
local SQLite cache so the next invocation can resume it. The cache
stores the message history client-side; on the next call mods replays
it as the conversation prefix.

It does **not** issue Anthropic `cache_control` markers or Gemini
`cachedContents.create` calls. For long conversations against
Anthropic, the lack of provider-side prefix caching is a real cost
hit; for short one-shot pipes (the design center), it doesn't matter.

## 8. Hot keybinds

There are none — there's no TUI. The interface is shell flags and
stdin/stdout. The closest things to "interactive" features:

| Flag | Action |
|------|--------|
| `mods <prompt>` | One-shot, stdout-only |
| `mods -f <prompt>` | Print formatted markdown (uses Glamour) |
| `mods -P <prompt>` | "Prompt" mode — no markdown rendering even on TTY |
| `mods -c <prompt>` | Continue the last saved conversation |
| `mods -C <name> <prompt>` | Continue / start a named conversation |
| `mods --list` | List saved conversations |
| `mods --show <name>` | Print a saved conversation |
| `mods --delete <name>` | Delete a saved conversation |
| `mods -m <alias>` | Pick a configured model alias |
| `mods --role <role>` | Apply a named system-prompt role from `mods.yml` |
| `mods --raw` | Skip Glamour rendering even when stdout is a TTY |

## 9. Killer feature, weakness, when to choose

- **Killer:** **it composes with everything in your shell.** Pipe
  `git diff | mods 'review this for bugs'`,
  `cat error.log | mods --role triage 'classify these'`,
  `find . -name '*.go' -newer yesterday | xargs -I{} mods 'summarize {}'`.
  The unix philosophy applied to LLMs. No other tool in this catalog
  does this as cleanly because they all want to own the session and
  the file edits.
- **Weakness:** **no agent loop, no tools, no file edits.** The model
  cannot read additional files it wasn't piped, cannot run shell
  commands, cannot retry-on-failure. Anything beyond "transform stdin
  to stdout" requires you to write the orchestration in shell.
- **Choose it when:** you want LLMs as a unix utility — for
  commit-message generation, log triage, doc summarization,
  one-off code review, ad hoc shell-pipeline reformatters, or
  scripted batch transforms. Choose something else when you need
  multi-step reasoning over a live repo.

## Pitfalls observed in real use

1. **Stdin must be finite.** `mods` reads stdin to EOF before sending.
   Don't pipe `tail -f` into it — it will buffer forever. Use a file
   or `head` to bound the input.
2. **Markdown rendering changes behavior in pipelines.** When stdout
   is a TTY, mods renders markdown via Glamour, which adds ANSI
   escape codes. When piping output to another program, pass `--raw`
   or pipe through `cat` to get plain text. The auto-detection is
   right 95% of the time and confusing the other 5%.
3. **Conversation cache is global.** `-C myname` collides across
   projects. There's no project-local namespace. Either prefix names
   with the project (`-C myrepo-debug`) or accept that the cache is
   a single user-wide pool.
4. **Token budget is whatever you set.** No automatic compaction, no
   summarization, no warning when you're about to overflow. Long
   `--continue` chains will eventually fail with a context-length
   error from the provider; you have to clear or fork the
   conversation manually.
5. **No retry on transient errors.** A 429 or a transient 5xx from
   the provider exits non-zero. Wrap in your own retry loop
   (`retry -n 3 -- mods ...` or shell `until` block) for batch
   workloads.
6. **Role definitions live in `mods.yml`, not version control by
   default.** If your team relies on a specific `triage` or `review`
   role, commit it to a shared dotfiles repo or vendor it into the
   project — otherwise role drift between machines is silent.
