# chatblade

> Snapshot date: 2026-04. Upstream: <https://github.com/npiv/chatblade>
> Binary name: `chatblade`

`chatblade` is the **structured-extraction Unix LLM utility** of the
catalog. Its tagline is "ChatGPT swiss army knife", and the framing
is accurate: it sits in the same niche as [`mods`](../mods/),
[`llm`](../llm/), and [`shell-gpt`](../shell-gpt/) — single-binary
text-in/text-out chat over OpenAI-compatible APIs — but its
distinguishing trick is what it does to the *response*.

`chatblade` can demand the model reply as JSON or YAML and then
**extract a sub-path with `jq`/`yq` syntax** in the same invocation,
without you parsing the response yourself. `chatblade -e '.commands[0]'
-j "list 3 git commands to undo my last commit"` returns a single
shell command, not a chat message — usable directly in a `$()`
substitution.

The other axis is **persistent named sessions on disk**, stored as
human-readable JSON / YAML files you can `git`-track, share, and
edit by hand. A session is just a list of message turns; you can
load one, append a new prompt, fork it, or replay it. This is
closer in spirit to [`llm`](../llm/)'s SQLite log than to
[`shell-gpt`](../shell-gpt/)'s in-memory chats, but the storage
shape (one file per session, plain text) is much easier to inspect,
edit, and version-control than a SQLite database.

The four canonical shapes:

- `chatblade "what's the capital of France"` — one-shot.
- `cat error.log | chatblade "what is causing these errors"` —
  pipe stdin in as the prompt body.
- `chatblade -s mysession "..."` — start or continue a named
  session (stored under `~/.config/chatblade/`).
- `chatblade -e '.steps[].command' -y "give me a yaml plan with
  steps"` — request YAML, extract a path with `yq`-like syntax.

There are also `-c gpt-4o` (model selector), `-l` (list
sessions), `-r` (raw, no system prompt), `--openai-api-key` /
`--openai-api-base` (point at a non-OpenAI endpoint), and `-t`
(estimate tokens before sending).

## 1. Install footprint

- Pure Python, ~200 KB plus dependencies (`openai`, `pyyaml`,
  `prompt_toolkit`, `rich`).
- `pipx install chatblade` (recommended), `uv tool install
  chatblade`, or `pip install chatblade` inside a venv.
- Config: `~/.config/chatblade/config.yaml` for defaults (model,
  temperature, system prompt, custom prompt aliases). Sessions
  live under `~/.config/chatblade/sessions/<name>.{json,yaml}`.
- Requires Python 3.10+. No native deps. Streams via the standard
  OpenAI SDK; works against any OpenAI-compatible base URL.

## 2. License

MIT.

## 3. Models supported

OpenAI built-in (the GPT-4 / GPT-4o / GPT-3.5 families and any
later model the upstream SDK exposes). Any other provider that
speaks the OpenAI HTTP shape works via `--openai-api-base` /
`OPENAI_API_BASE`: that includes Ollama, LM Studio, vLLM, LiteLLM
proxies, OpenRouter, Groq, Together, DeepSeek, Mistral's OpenAI-
compatible endpoint, and Azure OpenAI (with the appropriate
`api-version` query string).

There is no first-party Anthropic or Gemini path — for those you
either go through a LiteLLM proxy or pick a different catalog
entry ([`llm`](../llm/) with the appropriate plugin,
[`mods`](../mods/), [`aichat`](../aichat/) which has 20+ providers
built-in).

## 4. MCP support

None. No client, no server, no plugin slot. `chatblade` is a
single-purpose Unix utility; the extension story is "compose with
shell pipes", not "register an MCP tool". For MCP-driven workflows
use [`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`crush`](../crush/), or [`continue`](../continue/).

## 5. Sub-agent model

None. One invocation = one HTTP round-trip (or N if streaming).
There is no tool-call loop, no planner/builder split, no
recursion. The model writes a response, the response gets
extracted with the `-e` path, the result goes to stdout. That is
the entire control flow.

If you want the model to *do* something with the extracted result
(run the suggested command, edit a file), shell-pipe it onward:

```
chatblade -e '.commands[0]' -j "..." | sh
```

`chatblade` will not do this for you, by design.

## 6. Telemetry stance

**Off, with no opt-in.** No analytics calls. The only network
egress is the OpenAI (or OpenAI-compatible) endpoint you configured.
Logs of past sessions live on local disk in plain text — no daemon
syncs them anywhere. This is the "minimal trust surface" posture
shared with [`mods`](../mods/), [`llm`](../llm/),
[`shell-gpt`](../shell-gpt/), and [`fabric`](../fabric/).

If you point `--openai-api-base` at a third-party gateway
(OpenRouter, Together, etc.), that gateway sees the prompt — same
caveat that applies to every entry in the catalog.

## 7. Prompt-cache strategy

None client-side. `chatblade` does not maintain a prompt cache; it
sends the full session history on every continuation turn (same as
`shell-gpt`, `mods`, and the OpenAI SDK default). If your provider
(OpenAI 2024+, Anthropic via a proxy, etc.) supports server-side
prompt caching, it will be applied automatically — the
deterministic session-replay shape is friendly to prefix caching.

The `-t` flag estimates tokens locally with `tiktoken` before
sending, which is the right tool for "will this session fit in
the context window" budgeting.

## 8. Hot keybinds

There is no full TUI; the interactive surface is a `prompt_toolkit`
REPL when you run `chatblade -i` or continue an existing session.
The relevant ergonomics:

- `chatblade -i` — interactive multi-turn REPL, `Ctrl-D` to exit,
  `Ctrl-C` cancels generation.
- `chatblade -s work "..."` — name a session; future
  `chatblade -s work "..."` calls extend it.
- `chatblade -l` — list all sessions with size and last-modified
  time.
- `chatblade -s work --delete` — drop a session.
- `chatblade -e '.path.to.value' -j "prompt"` — JSON-extract.
- `chatblade -e '.path.to.value' -y "prompt"` — YAML-extract.
- `chatblade -r "prompt"` — raw mode, no system prompt prepended;
  useful when you want the model to behave like a plain completion
  endpoint.
- `chatblade -c gpt-4o-mini "prompt"` — per-call model override.
- `chatblade --prompt-config some_alias` — use a named prompt
  template from `config.yaml` (the closest thing `chatblade` has
  to [`fabric`](../fabric/)'s pattern library, but per-user and
  much smaller in scope).

## 9. Killer feature, weakness, when to choose

**Killer feature.** Inline structured-output extraction. The
combination of "ask the model for JSON/YAML" + "extract a sub-path
in the same command" + "exit code reflects whether the path was
found" makes `chatblade` uniquely shell-friendly for *programmatic*
use. You can write:

```
COMMAND=$(chatblade -e '.commands[0].cmd' -j \
  "give me a single bash command to find PDFs modified in the last week, \
   reply as JSON: {\"commands\": [{\"cmd\": \"...\"}]}") \
  && eval "$COMMAND"
```

…and it just works, with no `jq` pipe, no fragile regex on the
model's prose. No other entry in this catalog folds the response-
extraction step into the same invocation.

**Weakness.** Three:

1. **OpenAI-shape only.** Despite the `--openai-api-base` escape
   hatch, the assumed wire format is OpenAI chat-completions. If
   you live in the Anthropic or Gemini ecosystem and do not want
   a LiteLLM proxy, [`llm`](../llm/) (with plugins) or
   [`aichat`](../aichat/) (20+ native providers) is a better fit.
2. **No tool calls, no agent, no edits.** `chatblade` will never
   touch your filesystem and will never run a command. For "fix
   this failing test" workflows, use `aider`, `opencode`,
   `claude-code`, `codex`, or `cline`.
3. **Structured-extraction only as good as the model's compliance.**
   `-j` / `-y` rely on the model returning parseable JSON/YAML;
   on weaker models (or with a sloppy prompt) you will occasionally
   get prose-wrapped responses that the extractor fails on. Use
   GPT-4-class models for `-e` workflows, or fall back to
   "ask twice on parse failure" in your shell wrapper.

**When to choose.**

- **Shell scripts that need the model's output to be a value, not
  a paragraph.** `chatblade -e '.thing' -j` is the cleanest shape
  in the catalog for that.
- **You want chat sessions you can `cat`, `git diff`, hand-edit,
  and share.** Plain-text JSON/YAML session files beat
  [`llm`](../llm/)'s SQLite for "I want to read what I asked
  yesterday with `less`".
- **You want a `mods`-style pipe with persistent named sessions
  glued on**. `chatblade -s build_log "summarize" < build.log`
  appends both the input and the response to the named session
  file.
- **You are scripting against an OpenAI-compatible local model**
  (Ollama, vLLM, LM Studio) and you want JSON-mode extraction
  without writing your own HTTP client.

**When not to choose.**

- You need **MCP**. Use `opencode`, `claude-code`, `crush`,
  `continue`, `cline`.
- You need to **edit code**. Use `aider`, `opencode`, `codex`,
  `claude-code`, `cline`, `mentat`.
- You want **first-class non-OpenAI provider support without a
  proxy**. Use [`aichat`](../aichat/) or [`llm`](../llm/).
- You want a **shared, version-controlled prompt-pattern
  library**. Use [`fabric`](../fabric/); `chatblade`'s
  `prompt-config` aliases are per-user and much narrower in scope.
- You want **a queryable history of every call across every
  project**. Use [`llm`](../llm/) — its SQLite log is exactly
  that, and `chatblade`'s per-session files are not.
