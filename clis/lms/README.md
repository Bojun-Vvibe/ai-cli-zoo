# lms

> Snapshot date: 2026-04. Upstream: <https://github.com/lmstudio-ai/lms>

The **official command-line companion to LM Studio**, the
desktop app that downloads / runs / serves local LLMs on macOS,
Windows, and Linux. `lms` is the headless surface for everything
the GUI exposes: pulling models from HuggingFace, loading them
into the in-process llamacpp / MLX engine, listing what's
running, swapping the active model, exposing an
OpenAI-compatible HTTP server on `127.0.0.1:1234`, and tailing
inference logs — all from a script, an SSH session, or a CI job
that the LM Studio GUI cannot reach.

It is the catalog's reference for **a vendor-shipped local-model
control plane CLI** that pairs with the catalog's many CLIs that
expect an OpenAI-compatible endpoint
([`aichat`](../aichat/), [`aider`](../aider/),
[`shell-gpt`](../shell-gpt/), [`gptme`](../gptme/), etc.) but
need somewhere to point at — `lms` is the somewhere.

## 1. Install footprint

- LM Studio desktop app first (`brew install --cask lm-studio`
  on macOS, or the installer from `lmstudio.ai`); `lms` ships as
  part of that bundle and is bootstrapped onto `$PATH` via
  `~/.lmstudio/bin/lms bootstrap`.
- Standalone option: `npx lmstudio` proxies to the same binary
  for one-off use without a global install.
- Footprint: ~600 MB for the desktop app, ~50 MB for the CLI
  binary itself; model weights are downloaded on demand into
  `~/.lmstudio/models/`.
- Engines: bundled `llama.cpp` (GGUF, CPU + Metal / CUDA / ROCm /
  Vulkan), bundled `MLX` (Apple Silicon native), bundled `ONNX
  Runtime` (Windows / DirectML).

## 2. Repo, version, license

- Repo: <https://github.com/lmstudio-ai/lms>
- Version checked: **0.3.43** (from `package.json`; the project
  ships pre-releases via the LM Studio app rather than tagged
  GitHub releases).
- HEAD pinned at this snapshot:
  `0b2a176b4cb3e12564768ca6b67134a7b40e161c`.
- License: MIT. License file at
  [`LICENSE`](https://github.com/lmstudio-ai/lms/blob/main/LICENSE).

## 3. What it actually does

The CLI is verb-shaped around the LM Studio runtime:

```
lms ls                           # installed models + sizes
lms ps                           # currently loaded models
lms get qwen/qwen3-coder-30b     # download from HuggingFace mirror
lms load qwen3-coder-30b -y      # load into engine, allocate KV cache
lms unload qwen3-coder-30b
lms server start                 # OpenAI-compatible API on :1234
lms server status
lms log stream                   # tail live inference logs
lms chat                         # interactive REPL against loaded model
```

The launched server speaks a superset of the OpenAI
`/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, and
`/v1/models` shape, plus an `lmstudio` JSON-RPC channel for
structured streaming + KV-cache control. Any catalog CLI that
accepts `OPENAI_BASE_URL=http://127.0.0.1:1234/v1` works against
it without code changes.

The `lms load` step is the interesting one: it actually
allocates GPU / unified memory, warms the KV cache, and
optionally pins a draft-model for speculative decoding (`lms load
big-model --draft small-model`). This is the speedup
[`aichat`](../aichat/) / [`aider`](../aider/) get for free when
they hit the LM Studio endpoint vs. a cold `llamafile` start.

## 4. MCP support

None first-party in the CLI. The LM Studio desktop app added
client-side MCP in late 2025 for the GUI chat surface; the CLI
inherits that only via the `lms server` HTTP endpoint, which
forwards function-calling shaped requests to the loaded model.
There is no `lms mcp` subcommand.

## 5. Sub-agent model

None — `lms` is a runtime / server controller, not an agent
loop. Multi-model orchestration ("planner on big model, editor
on small model") happens at the consumer side
([`forge`](../forge/), [`crewai`](../crewai/),
[`langgraph`](../langgraph/)) which point at the same `:1234`
endpoint with different `model:` ids and let LM Studio swap them
on demand if `lms server --auto-load` is on.

## 6. Telemetry stance

Off. The CLI and the bundled server do not phone home; the LM
Studio desktop app has an opt-in update-check ping. All
inference happens locally; egress is only the HuggingFace
`/api/models` calls that `lms get` makes when downloading new
weights.

## 7. Token / context strategy

Per-model, configured at load time:
`lms load <model> --context-length 32768 --gpu-layers 999
--flash-attn`. Context length is a hard cap enforced by the
underlying engine; `lms` exposes it as a CLI flag rather than
asking the consumer CLI to negotiate it. Multiple models can be
loaded simultaneously up to memory limits, each with its own
KV cache; the OpenAI-compatible server routes requests by the
`model:` field in the request body.

## 8. Hot keybinds

`lms chat` is a simple line-editor REPL: `↑` recalls history,
`Ctrl-C` cancels the current generation (model keeps its KV
cache), `Ctrl-D` exits. There is no fullscreen TUI; for a richer
local-Ollama-style TUI against the same endpoint, pair with
[`oterm`](../oterm/) or [`aichat`](../aichat/).

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A first-party headless control plane for
LM Studio that is just a CLI.** The desktop app's strength
(GUI-driven model browser, easy GGUF downloads, working
Metal / CUDA / ROCm engine selection, OpenAI-compatible server
that "just works" for hundreds of consumer CLIs) becomes
scriptable: a CI job can `lms get`, `lms load`, run an eval
suite via [`promptfoo`](../promptfoo/), `lms unload`, exit. SSH
into a remote workstation, `lms server start`, and tunnel
`:1234` back to a laptop CLI without ever touching a GUI.

**Weakness.** Couples to the LM Studio desktop app — `lms` is
not a standalone runtime, it is a remote control. If the desktop
app is uninstalled, `lms` breaks; if you want a pure
headless-from-day-one local server,
[`ollama`](../ollama/) / [`llamafile`](../llamafile/) /
[`vllm`](../vllm/) / [`ramalama`](../ramalama/) are better fits.
Closed-source app + open-source CLI is an unusual split — the
CLI is MIT, the inference engine bundle is not, so air-gapped /
license-strict environments need to audit both. Limited Linux
support relative to macOS / Windows; some engine builds lag the
desktop release.

**Choose `lms` when** LM Studio is already the local-model
runtime (developer laptops, mixed-OS team) and you want the same
GUI-curated model library scriptable from CI and SSH. **Choose
something else when** you need a daemon-first runtime
([`ollama`](../ollama/)), a single-binary self-contained model
([`llamafile`](../llamafile/)), high-throughput batched serving
([`vllm`](../vllm/) / [`sglang`](../sglang/)), or a fully
open-source stack end-to-end ([`ramalama`](../ramalama/)).
