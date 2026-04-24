# tlm

> Snapshot date: 2026-04. Upstream: <https://github.com/yusufcanb/tlm>
> Binary name: `tlm`

`tlm` is a **local-only shell helper** that talks exclusively to an
Ollama daemon and answers three narrow questions: *"what's the
shell command for X?"*, *"what does this command/file/diff mean?"*,
and *"chat with me in the terminal"*. No cloud provider toggles,
no API key field, no agent loop, no file edits — just a Go binary
that pairs a tiny prompt-template library with a local LLM and a
nice TTY confirmation prompt.

```
ollama serve &
ollama pull qwen2.5-coder:3b
tlm install            # wires `tlm` to the local Ollama, picks a model
tlm suggest "find all .log files larger than 100MB modified this week"
```

The output is a single shell command rendered with syntax color and
a `[E]xecute / [C]opy / [A]bort` prompt. That's the whole product.

This is taxonomically distinct from [`shell-genie`](../shell-genie/)
(Python, OpenAI by default, hosted free endpoint as fallback),
[`shell-gpt`](../shell-gpt/) (Python, OpenAI-compatible, broader
chat surface), and [`gorilla-cli`](../gorilla-cli/) (hosted
Gorilla endpoint, multi-candidate picker). `tlm`'s differentiator
is **single Go binary, Ollama-only, zero network egress, with a
`tlm install` bootstrap that pulls a model for you**.

## 1. Install footprint

- Single static Go binary, ~20 MB. macOS / Linux / Windows builds
  on every release.
- Install via the upstream installer:
  - `curl -fsSL https://raw.githubusercontent.com/yusufcanb/tlm/main/install.sh | sudo -E bash`
  - or download the release binary from GitHub and drop it on
    `$PATH`.
- Requires Ollama reachable at the URL `tlm install` records
  (default `http://localhost:11434`). The `install` subcommand
  probes the daemon, lets you pick which Ollama model to use for
  `suggest` vs `explain`, and writes `~/.tlm.yml`.
- No Python, no Node, no cgo, no system libraries beyond libc.
  Works fine in minimal container images.

## 2. License

Apache-2.0.

## 3. Models supported

**Ollama only.** Whatever models `ollama list` reports on the
configured endpoint. The recommended defaults (suggested by `tlm
install`) are small coder-tuned models that respond in a few
hundred milliseconds on consumer hardware: `qwen2.5-coder:3b`,
`llama3.2:3b`, `phi3:3.8b`. Larger models (Qwen 32B, Llama 3.3 70B)
work too if your box can run them.

There is no OpenAI / Anthropic / Gemini support and no plan to add
it — `tlm` is explicitly a local-first tool. If you want a
provider-agnostic shell helper, the catalog has
[`shell-gpt`](../shell-gpt/), [`mods`](../mods/), and
[`tgpt`](../tgpt/).

## 4. MCP support

**No.** `tlm` is a one-shot prompt-template runner. There is no
tool-call loop, no MCP client, no plugin host. Each invocation is
a single round trip to Ollama with a baked-in system prompt
appropriate to the subcommand (`suggest`, `explain`, `chat`).

## 5. Sub-agent model

**None.** Same reason as §4 — `tlm` is not an agent. It is a
two-screen interaction: ask, then `[E]xecute / [C]opy / [A]bort`.

## 6. Telemetry stance

**Off, no opt-in path.** `tlm` itself does not phone home. The only
network traffic is the HTTP call to your configured Ollama
endpoint. No analytics, no crash reporter, no update check. If
you point `tlm` at a remote Ollama you control, that endpoint is
the only place prompts go.

## 7. Prompt-cache strategy

Inherits Ollama's KV cache. Because `tlm suggest` invocations are
all short and use the same system prompt, Ollama's prefix cache
hits cleanly across runs as long as the same model stays loaded.
There is no `tlm`-side history, no completion cache, no
deduplication. `tlm chat` keeps a single in-memory transcript for
the duration of the chat session and discards it on exit.

## 8. Hot keybinds / surface

`tlm` has three subcommands and one bootstrap:

- `tlm install` — interactive setup; probes Ollama, lets you pick
  the suggest-model and explain-model, writes `~/.tlm.yml`.
- `tlm suggest "<task>"` (alias `tlm s`) — natural-language → one
  shell command, prompts `[E]xecute / [C]opy / [A]bort`.
- `tlm explain "<command>"` (alias `tlm e`) — explain a shell
  command in plain English; also accepts piped input
  (`cat error.log | tlm explain`).
- `tlm chat` (alias `tlm c`) — interactive REPL with the
  configured chat model. `Ctrl+C` to exit, `/clear` to wipe
  context.
- `tlm config` — print the resolved `~/.tlm.yml`.

There are no TUI keybinds beyond stdin/stdout — `tlm` is a CLI,
not a Textual app. The execute/copy/abort prompt accepts
single-key responses.

## 9. Killer feature, weakness, when to choose

**Killer feature.** A **single static Go binary** that delivers
the natural-language → shell-command loop **with zero cloud
dependencies**, plus a `tlm install` bootstrap that pulls and
configures an Ollama model in one command. The other shell helpers
in the catalog either need Python (`shell-gpt`, `shell-genie`),
default to a hosted endpoint that sees your prompts (`tgpt`,
`gorilla-cli`, `shell-genie`'s free-genie), or are general
chat tools that can answer shell questions but were not built for
it. `tlm` is the smallest installable surface for the local-only
"give me one command, ask before running it" workflow.

**Weakness.** Three:

1. **Ollama-only.** Cannot use OpenAI, Anthropic, or any other
   cloud provider. If you need cloud quality, `shell-gpt` is the
   right pick.
2. **Quality is bounded by your local model.** A 3B coder model
   gets routine `find` / `awk` / `grep` invocations right but
   hallucinates obscure flags and made-up subcommands. Verify
   before pressing `E`.
3. **No multi-candidate UX.** Unlike [`gorilla-cli`](../gorilla-cli/),
   `tlm suggest` returns one answer; you regenerate to see another.

**When to choose.**

- You already run **Ollama locally** and want a one-binary helper
  that respects that.
- You are on an **air-gapped or privacy-sensitive box** where
  shipping prompts to OpenAI is not allowed.
- You want **native binaries** (no Python venv, no Node, no `pipx`)
  for shipping `tlm` to coworkers or container images.
- You want **`tlm install` to pick the model for you** instead of
  reading three READMEs to figure out a sane Ollama default.

**When not to choose.**

- You need **cloud-quality answers** for unusual tooling. Use
  [`shell-gpt`](../shell-gpt/) with GPT-4o or Claude.
- You want the model to **see multiple candidates** and let you
  arrow-key between them. Use [`gorilla-cli`](../gorilla-cli/).
- You want a tool that **edits files** based on the conversation.
  `tlm` only suggests-and-runs; reach for [`aider`](../aider/) or
  [`opencode`](../opencode/) for that.
- You want **sample command generation across many languages** in
  the same TUI. `tlm chat` is intentionally minimal; pick
  [`oterm`](../oterm/) for a real Ollama TUI with multi-tabs and
  vision.

## Sample command + expected shape

```
$ tlm suggest "find every .log file under /var/log larger than 100M modified this week"
► find /var/log -type f -name "*.log" -size +100M -mtime -7
[E]xecute  [C]opy  [A]bort: c
copied to clipboard.

$ tlm explain "tar --use-compress-program='zstd -T0 -19' -cf out.tar.zst dir/"
Creates out.tar.zst, a tar archive of `dir/` compressed with zstd at
level 19 using all available CPU cores (-T0). Higher level → smaller
file, slower. zstd is faster than xz at comparable ratios.
```

## Links

- Source: <https://github.com/yusufcanb/tlm>
- Releases: <https://github.com/yusufcanb/tlm/releases>
- Ollama (required backend): <https://ollama.com>
