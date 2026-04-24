# smartcat (sc)

> Snapshot date: 2026-04. Upstream: <https://github.com/efugier/smartcat>
> Binary name: `sc`

`smartcat` is a **Unix-pipe LLM `cat`**: it reads stdin, prepends
a configurable prompt, sends it to a configured backend, and
streams the model's reply to stdout. The whole point is to behave
like a well-mannered Unix filter so you can chain it with `grep`,
`jq`, `awk`, `xargs`, and friends without thinking about it.

```
cat src/parser.rs | sc "explain the error path in this file" > notes.md
git diff | sc -p commit "write a conventional commit message" | tee msg.txt
echo "list the 7 layers of the OSI model as JSON" | sc | jq .
```

The binary name is `sc` (not `smartcat`) and that is on purpose:
it sits next to `cat`, `grep`, `sed`, `awk` in your muscle memory
and reads naturally in pipelines.

This is taxonomically next to [`mods`](../mods/) (charm.sh, Go,
OpenAI / Anthropic / Gemini / Ollama / OpenAI-compatible),
[`llm`](../llm/) (Python, every prompt logged to SQLite),
[`fabric`](../fabric/) (curated pattern library), and
[`tgpt`](../tgpt/) (free no-API-key cloud endpoints by default).
`smartcat`'s differentiator is a **single Rust binary with a
prompts-as-config-file model** (`~/.config/smartcat/prompts.toml`),
a **`-p <name>` flag** that swaps the system prompt without
rewriting the pipeline, and **first-class glob input**
(`sc -g 'src/**/*.rs' "find dead code"`).

## 1. Install footprint

- Single static Rust binary. Install with `cargo install
  smartcat`, `brew install smartcat` (Homebrew tap), or download
  a release binary from GitHub.
- Config files live at `~/.config/smartcat/`:
  - `.api_configs.toml` — provider endpoints + API keys (one
    block per provider; `smartcat` does not invent its own
    secret store).
  - `prompts.toml` — named prompts, each with optional model
    pin, temperature, and a list of pre-canned messages.
  - `conversation.toml` — auto-written transcript of the last
    pipeline that used `-e/--extend-conversation`.
- No Python, no Node, no system libs beyond libc and OpenSSL (or
  rustls on the rustls-built artifact).
- First run prints the path of the config files it just bootstrapped
  and exits with a friendly "fill in your API key" message.

## 2. License

Apache-2.0.

## 3. Models supported

**Any HTTP-reachable provider** declared in `.api_configs.toml`.
Out of the box the config template ships blocks for OpenAI,
Anthropic, Mistral, Groq, and a generic OpenAI-compatible block
that covers Ollama, vLLM, LM Studio, LiteLLM proxies, OpenRouter,
DeepSeek, Together, etc. Each prompt in `prompts.toml` can pin a
specific provider+model, so you can have `sc -p commit` use a
fast/cheap model and `sc -p review` use a frontier model from the
same shell.

There is no built-in provider abstraction beyond "speak the
provider's documented HTTP shape" — you copy the block, paste your
key, set the model name, you are done.

## 4. MCP support

**No.** `smartcat` is a stdin-in / stdout-out filter. There is no
tool-call loop, no MCP client, no plugin host. The prompts file
is the only extension surface. If you need tool calls in a Unix
pipeline, [`gptscript`](../gptscript/) is the catalog answer.

## 5. Sub-agent model

**None.** Each `sc` invocation is one HTTP call. The
`-e/--extend-conversation` flag re-uses the last
`conversation.toml` so the next call sees prior turns, but that
is shell-driven persistence, not multi-agent orchestration.

## 6. Telemetry stance

**Off, no opt-in.** `smartcat` itself sends nothing. Network
traffic is exclusively the HTTPS call to whatever provider
endpoint the resolved prompt points at. The conversation file is
local; no analytics, no crash reporter, no update ping.

## 7. Prompt-cache strategy

`smartcat` does not cache. It does, however, support
**conversation extension** (`-e`) which lets the *provider*'s
prompt-cache work: the same prefix (system prompt + prior turns)
gets resent on the next call, so providers with prefix caching
(Anthropic, Gemini, OpenAI on certain models) reuse their cache
naturally. There is no client-side response cache, no prompt
deduplication. The `--temperature` and `--n-completions` flags
drive provider-side determinism instead.

## 8. Hot keybinds / surface

`smartcat` is a one-shot CLI; there are no TUI keybinds. The
flags that matter:

- `sc "<extra prompt>"` — read stdin, prepend the default prompt
  *plus* this extra string, send, stream stdout.
- `sc -p <name>` — use the named prompt block from
  `~/.config/smartcat/prompts.toml`.
- `sc -e` — extend the prior conversation (multi-turn from the
  shell).
- `sc -c` — copy the *output* to the system clipboard in
  addition to printing it.
- `sc -i <file>` — prepend the contents of `<file>` to the
  prompt (useful when stdin is reserved for something else).
- `sc -g '<glob>'` — read every file matching a glob and use
  it as input. `sc -g 'src/**/*.rs' "find dead code"` packs an
  entire crate.
- `sc --api <name> --model <name>` — override the prompt's
  pinned provider/model for one call.
- `sc -l` — list the named prompts in `prompts.toml`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Named prompts as a config file**, swappable
with one flag. `sc -p commit < diff.txt` and `sc -p review <
diff.txt` pull *different* system prompts, *different* model pins,
*different* temperature settings — without changing the pipeline.
Combine with `-g` for glob input and `-e` for shell-driven
multi-turn, and you have a Unix-native LLM filter that scales from
"explain this stack trace" to "review every changed file in this
PR" without leaving the shell.

The closest cousin is [`mods`](../mods/), which also pipes
beautifully and supports named "roles". `smartcat` is the right
pick when you prefer **Rust + a single TOML file** over Go + a
YAML + per-flag overrides, and when the **glob input** flag
matters to you (`mods` has no equivalent of `sc -g`).

**Weakness.** Three:

1. **No tool calls, no agency.** This is not an agent; it is a
   filter. For autonomous repo edits, switch to
   [`aider`](../aider/) or [`opencode`](../opencode/).
2. **Provider config is hand-edited TOML.** Easier to grok than
   environment-variable sprawl, but you do edit a file to add a
   new provider; there is no `sc auth login`.
3. **No prompt history.** Unlike [`llm`](../llm/), `smartcat`
   does not log every call to SQLite. If you want replayability,
   pipe through `tee` yourself or use `llm` as your stdin/stdout
   driver.

**When to choose.**

- You **live in pipelines** and want a clean Unix filter that
  respects stdin/stdout/stderr, exit codes, and `set -e`.
- You want **multiple system prompts in a config file** that you
  can `git`-track, share with the team, and swap with `-p <name>`.
- You want a **single Rust binary** with no Python/Node footprint
  for shipping to dev containers or CI runners.
- You need **glob input** (`sc -g 'src/**/*.rs' "find dead
  code"`) without writing a packer-and-pipe boilerplate.

**When not to choose.**

- You need an **agent that runs tools and edits files**. Use
  [`aider`](../aider/), [`opencode`](../opencode/), or
  [`claude-code`](../claude-code/).
- You want **every prompt logged to SQLite** for later replay.
  Use [`llm`](../llm/).
- You want a **curated, version-controlled prompt-pattern
  library** maintained by a community. Use
  [`fabric`](../fabric/).
- You want a tool that **runs with no API key out of the box**.
  Use [`tgpt`](../tgpt/) (free cloud endpoints) or
  [`tlm`](../tlm/) (local Ollama).

## Sample command + expected shape

```
$ git diff --staged | sc -p commit
feat(parser): tighten error path on truncated UTF-8 input

The previous decoder returned `Ok(String::new())` on a half-character
boundary; callers downstream interpreted "" as a successful empty
parse. Return a typed `ParseError::TruncatedUtf8 { offset }` so the
retry loop in `read_chunk` can resync to the next code-point start.

$ sc -g 'src/**/*.rs' "list every public function that does file I/O without an explicit error type" \
    | tee audit.md
src/cache.rs::load_index            -> uses anyhow::Result, not a typed error
src/loader.rs::scan_dir             -> returns ()? io errors are silently logged
src/loader.rs::write_manifest       -> returns std::io::Result, OK
...
```

## Links

- Source: <https://github.com/efugier/smartcat>
- Crate: <https://crates.io/crates/smartcat>
- Config docs: <https://github.com/efugier/smartcat#configuration>
