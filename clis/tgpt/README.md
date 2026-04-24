# tgpt

> Snapshot date: 2026-04. Upstream: <https://github.com/aandrew-me/tgpt>
> Binary name: `tgpt`

`tgpt` ("Terminal GPT") is the **no-API-key primitive** of the catalog.
Every other entry in this repo assumes you have an account and a key
somewhere — OpenAI, Anthropic, Gemini, OpenRouter, a local Ollama at
the very least. `tgpt` ships with a curated list of free public
inference endpoints baked into the binary and selects one of them by
default. `brew install tgpt && tgpt "what does ENOSPC mean"` works
zero-config, end of setup.

That single design decision is what makes `tgpt` a distinct entry
rather than "just another `sgpt` clone". The rest of the surface
looks similar — single static Go binary, one-shot prompt-to-stdout,
optional interactive session, optional shell-command generation —
but the contract with the user is different: **the default provider
list is the product**.

Three modes define the surface:

- `tgpt "explain rsync --info=progress2"` — one-shot, stream to stdout.
- `tgpt -i` — interactive mode (multi-turn, no history persistence
  across runs unless you pass `--session <name>`).
- `tgpt -s "find all PDFs modified this week"` — shell-command
  generator with a `Choose: 1) Execute  2) Modify  3) Copy  4) Cancel`
  prompt afterward.

There are also `-img` (image generation), `-q` (quiet, no animation),
and `-w` (whole-shot, no streaming). The `-cl` flag dumps a code-only
response with no fences, suitable for piping into a file.

## 1. Install footprint

- Single static Go binary, ~10 MB. No runtime, no venv, no Node.
- `brew install tgpt`, `curl -sSL https://raw.githubusercontent.com/aandrew-me/tgpt/main/install | bash` (writes to `/usr/local/bin`), or `go install github.com/aandrew-me/tgpt/v2@latest`.
- Config (provider, model, temperature, system prompt) lives at
  `~/.config/tgpt/config.toml` if you want to override defaults.
  The vast majority of users never touch it.
- No dependencies on Python, Node, or any provider SDK at the binary
  level — the free providers are all reverse-engineered HTTP endpoints
  spoken from Go directly.

## 2. License

GPL-3.0.

## 3. Models supported

This is where `tgpt` is taxonomically odd. Built-in providers (no key,
no signup) cycle through a curated list that has shifted over time
but currently includes:

- **Pollinations** (default) — proxies multiple OpenAI-compatible
  free endpoints; selects a backend per request.
- **Phind** — Phind-70B and various open-weights served free.
- **Blackbox AI** — free chat backend.
- **DuckDuckGo AI Chat** — free, anonymous, rate-limited.
- **Koboldai** — community-hosted open-weights.

You can also override with **bring-your-own-key** providers via
`--provider openai|anthropic|groq|ollama|openrouter` and the matching
env var. So the ladder is:

1. **No key** → free providers (default).
2. **Free-tier key** (Groq, OpenRouter free models, Cerebras free) →
   `--provider groq --model llama-3.3-70b-versatile`.
3. **Paid key** (OpenAI/Anthropic) → same flag pattern, normal billing.
4. **Local** → `--provider ollama --model qwen2.5-coder:7b`.

The free-provider story is the headline; everything else is
"`tgpt` can also do what `sgpt`/`mods` do".

## 4. MCP support

None. No client, no server, no plugin slot. `tgpt` is deliberately a
single static binary with no extension surface — the philosophy is
"if you want plugins, install one of the other 22 entries in this
catalog." It is a primitive, not a platform.

## 5. Sub-agent model

None. Each invocation is one HTTP request (or N if you are inside
`-i`/`--session` and the model loops on its own). There is no
planner/builder split, no tool-call loop, no recursion. The shell
mode (`-s`) does exactly one round-trip: prompt → command → choose.
There is no "the model ran the command, saw it failed, and tried
again" loop. That is intentional: the project's goal is "smallest
possible LLM-in-terminal", not "agent runtime".

## 6. Telemetry stance

**Off, with no opt-in.** `tgpt` itself sends nothing. The free
providers it talks to obviously see prompts — that is the price of
"no key" — and the upstream README is explicit about this: assume
free-provider prompts are logged and possibly used for training,
treat them as you would any anonymous web search. For sensitive
prompts, switch to `--provider ollama` (local) or to a paid provider
with a written data policy.

## 7. Prompt-cache strategy

None. There is no on-device cache, no prompt-prefix reuse, no
session-warm trick. Each call is stateless from the provider's
perspective. Inside `-i`/`--session`, prior turns are re-sent each
time, same as every other chat-style CLI.

The free providers do not advertise prompt caching, so even
provider-side prefix caching cannot be relied on. If you care about
caching latency or token cost, you have already left the free-tier
contract — switch to a paid provider where caching is documented.

## 8. Hot keybinds

There is no TUI, just an interactive REPL (`-i`). Relevant ergonomics:

- `tgpt -i` — interactive REPL. `Ctrl-D` exits, `Ctrl-C` cancels the
  current generation but stays in the REPL.
- `tgpt --session work` — same as `-i` but persists the conversation
  to `~/.config/tgpt/sessions/work.json`. Resume across shell sessions.
- After `tgpt -s "..."` — the four-way choice prompt:
  `1) Execute  2) Modify  3) Copy to clipboard  4) Cancel`.
  `2) Modify` drops the generated command into your `$EDITOR` for an
  edit pass before execution; this is more conservative than
  `shell-gpt`'s three-way `[E/D/A]`.
- `tgpt -ip "describe the prompt"` — image generation; output writes
  a PNG to `./generated_image.png` and prints the path.
- `tgpt --whole "..."` — disable streaming; useful when piping
  `tgpt --whole "..." | jq` and you need a single complete blob.

There is no `Ctrl-L` shell-buffer integration like `shell-gpt`; if
you want that ergonomic, layer it yourself with a shell function.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Zero-config "LLM in my terminal right now". The
moment from `brew install tgpt` to a useful answer is about 30
seconds, with no signup, no key, no quota dashboard, no credit card.
For demos, throwaway VMs, classroom settings, fresh laptops where you
want to answer "is the LLM thing useful?" before investing in
plumbing, nothing else in the catalog is close — every other entry
requires *some* account somewhere.

**Weakness.** Three big ones:

1. **Free providers are unstable.** Endpoints break, get rate-limited,
   get pulled by the upstream service, or silently return degraded
   responses. The `tgpt` maintainers do their best to keep the
   provider list current, but a release that worked last month may
   route through a dead backend today. If you depend on `tgpt` in a
   script, pin to a paid `--provider` you control.
2. **No project context, no tools, no edits.** `tgpt` does not look
   at your repo, your shell history, your environment beyond the
   prompt you typed. It cannot edit files. It is strictly a
   text-in/text-out (and shell-command-out) primitive. For "fix this
   failing test", use anything else in the catalog.
3. **GPL-3.0.** Strongly copyleft — fine for personal use, but if
   you are integrating `tgpt` into a closed-source distribution, the
   licensing math is not the same as `sgpt` (MIT) or `mods` (MIT).
   Most users will never hit this; teams shipping commercial CLIs
   that bundle `tgpt` will.

**When to choose.**

- **Zero-friction demo.** You want to show a non-technical colleague
  "this is what an LLM in a terminal looks like" without setting up
  an account first.
- **Fresh / disposable environments.** A VM, a container, a
  classroom Chromebook, a borrowed laptop where you do not want to
  type API keys.
- **`mods`-style pipe usage with no key.** `cat error.log | tgpt
  --whole "what is causing these errors?"` works the same shape
  as `mods`, without needing `OPENAI_API_KEY` set.
- **Throwaway shell-command generation** for one-off Unix flag
  questions where you do not care which model answers, you just want
  the command.
- **Privacy-by-locality** via `--provider ollama` once you outgrow
  the free providers — `tgpt` makes a clean migration path from
  "no key" to "local model" without changing the invocation shape.

**When not to choose.**

- You need **reliable, reproducible** output for a script that runs
  in CI. The free-provider routing is too volatile. Use `mods` or
  `llm` against a paid endpoint.
- You need to **edit project files**. Use `aider`, `opencode`,
  `codex`, `claude-code`, or `cline`.
- You need **MCP** integration. `tgpt` does not have it.
- You need a **queryable log** of every call. Use [`llm`](../llm/),
  whose SQLite log is exactly that.
- You need **`Ctrl-L` shell integration** specifically — that is
  [`shell-gpt`](../shell-gpt/)'s thing, not `tgpt`'s.
- You are **shipping a closed-source product** that bundles a
  terminal LLM. GPL-3.0 will bite you; pick an MIT/Apache-2.0 entry.
