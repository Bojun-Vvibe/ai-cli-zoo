# ramalama

> Snapshot date: 2026-04. Upstream: <https://github.com/containers/ramalama>
> Binary name: `ramalama`

`ramalama` is a **container-native local-model runner** in the
spirit of Ollama, but built on top of OCI containers (Podman or
Docker) instead of a custom daemon. It pulls models from registries
(Hugging Face, Ollama's registry, OCI artifact registries),
launches them inside a hardware-appropriate container image
(`llama.cpp` CPU, CUDA, ROCm, Vulkan, Intel oneAPI, Apple Metal),
exposes an OpenAI-compatible HTTP server on demand, and provides
a `chat`-style REPL on top of all of it.

```
pip install ramalama
ramalama pull granite3-dense:8b
ramalama run granite3-dense:8b
> /set system "be terse"
> count the planets

ramalama serve --port 11434 granite3-dense:8b
# -> OpenAI-compatible endpoint at http://localhost:11434/v1
```

The crucial property: every model run happens inside an
auto-selected container image, so there is no host-level
`llama.cpp` install to maintain, no CUDA driver dependency
mismatch, and no surprise GPU fallback. If your box has CUDA
12.x, `ramalama` runs the CUDA image; if not, it runs the CPU
image; if you have an AMD card, it runs the ROCm image. Same
command line, same OpenAI-compatible API surface.

This is taxonomically distinct from Ollama (custom daemon, custom
registry, hides the inference engine), from `llama.cpp`'s `llama-cli`
(no model registry, no API server bootstrap), and from
[`oterm`](../oterm/) (a UI on top of a separate Ollama daemon).
`ramalama` is the **runtime** layer; pair it with `oterm`, `mods`,
`aichat`, `aider`, or any OpenAI-compatible client as the UI.

## 1. Install footprint

- Python package on PyPI: `pip install ramalama` or `pipx install
  ramalama`. Available in Fedora as `dnf install ramalama`.
- Hard dependency: a working **Podman** or **Docker** install on
  the host. `ramalama` will refuse to start if neither is
  reachable; on macOS that means `podman machine init && podman
  machine start` first (or Docker Desktop).
- The first `pull` of a given hardware family triggers a one-time
  container-image pull (usually 1-3 GB). Subsequent runs reuse the
  cached image.
- Models live in `~/.local/share/ramalama/models/` by default
  (configurable via `--store`), as plain GGUF files plus a small
  manifest. Models pulled from Ollama's registry are deduplicated
  with Ollama's local store when both are present.
- No Node, no Go toolchain. Pure Python at the entry-point layer;
  the heavy lifting happens inside the container.

## 2. License

Apache-2.0. (Note: the *container images* it pulls inherit their
own licenses — `llama.cpp` is MIT, CUDA images carry the NVIDIA
license, etc. `ramalama` itself is Apache-2.0.)

## 3. Models supported

Any **GGUF** model `llama.cpp` can load. Sources, in order of
discovery convenience:

- `ramalama pull ollama://granite3-dense:8b` — pull from Ollama's
  registry (the `ollama://` scheme is the default if you omit a
  scheme).
- `ramalama pull huggingface://TheBloke/Mistral-7B-Instruct-v0.2-GGUF` —
  pull a HF repo, picks a sensible quantization by default.
- `ramalama pull oci://quay.io/ai-lab/granite-7b-lab` — pull an
  OCI artifact (the project's preferred long-term distribution
  shape; models become `podman pull`-able like images).
- `ramalama pull file:///path/to/local.gguf` — register a local
  file.

There is no first-class support for closed-weight cloud models
(OpenAI, Anthropic, Gemini). The intent is the inverse — give you
a *local* OpenAI-compatible endpoint that other CLIs in this
catalog can point at instead of OpenAI.

## 4. MCP support

**No** native MCP client. `ramalama serve` exposes an
OpenAI-compatible chat-completions endpoint; MCP-aware clients
(`opencode`, `claude-code`, `cline`, `continue`, etc.) call the
endpoint directly and run their own MCP servers on top.

## 5. Sub-agent model

**None.** `ramalama` is the runtime, not the agent layer. The
chat REPL it ships is a single-turn-into-context loop with no
tool calls. Multi-agent orchestration is the consumer's job —
point [`opencode`](../opencode/), [`continue`](../continue/), or
[`aider`](../aider/) at `http://localhost:<port>/v1` and let those
tools do the agenting.

## 6. Telemetry stance

**Off, no opt-in.** `ramalama` does not phone home. The container
runtime (Podman / Docker) has its own telemetry posture; that's
on the host operator. The OCI image pulls and the Hugging Face /
Ollama-registry pulls are visible to those upstreams the same way
any `podman pull` or `huggingface-cli download` is.

## 7. Prompt-cache strategy

Inherits `llama.cpp`'s KV-cache behavior. Within a single
`ramalama serve` process, the same chat session keeps its KV
cache warm; switching models triggers a container restart and a
cold cache. There is no `ramalama`-level disk cache of completions.
The `--ctx-size` flag passes through to `llama.cpp`'s `-c` and
trades RAM for context length.

## 8. Hot keybinds / surface

`ramalama` is mostly subcommands; the chat REPL is intentionally
minimal:

- `ramalama list` — show locally pulled models.
- `ramalama pull <ref>` — pull a model (any of the schemes in
  §3).
- `ramalama run <ref>` — interactive chat REPL. Slash commands:
  - `/set system "<text>"` — set the system prompt.
  - `/clear` — wipe the conversation.
  - `/show info` — print the model + container being used.
  - `/bye` (or `Ctrl+D`) — quit.
- `ramalama serve [--port N] <ref>` — start an OpenAI-compatible
  HTTP server backed by the model. Default port `8080`; pick
  `11434` to be a drop-in for Ollama clients.
- `ramalama stop <ref>` — stop a `serve` container.
- `ramalama rm <ref>` — delete a local model.
- `ramalama bench <ref>` — run `llama-bench` inside the container
  for quick tokens-per-second numbers.
- `ramalama containers` — list `ramalama`-launched containers.
- `ramalama info` — print resolved settings (chosen image family,
  store path, container engine).

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Hardware-aware container selection** for
local LLM inference. `ramalama` looks at the host (CUDA major
version, ROCm presence, Vulkan, Apple Metal, plain CPU) and picks
the matching `llama.cpp` image automatically. Same command line
on a Linux laptop with no GPU, a workstation with an NVIDIA
4090, an AMD MI300, and an Apple M-series Mac. Combine with
`oci://` distribution and you can ship the *exact* model + runtime
combo through the same registry plumbing as your application
images, instead of curating a `Modelfile` and praying the host
has the right CUDA. This is the property no other entry in this
catalog has.

**Weakness.** Three:

1. **Container engine is mandatory.** No host-mode fallback; if
   you cannot run Podman or Docker on the box, `ramalama` is the
   wrong tool. Reach for direct `llama.cpp`, Ollama, or
   `llamafile` instead.
2. **Cold-start cost.** First run on a new hardware family pulls
   a multi-GB image. Worth it on a workstation; painful on a
   tiny CI runner.
3. **Not an agent or UI.** The chat REPL is bare; the value is
   the runtime + API, not the user surface. Pair with another
   catalog entry for ergonomics.

**When to choose.**

- You want a **local, OpenAI-compatible endpoint** that other
  CLIs in this catalog can target without OpenAI keys.
- You want **the same `pull`/`run` UX across CPU, NVIDIA, AMD,
  Intel, and Apple Silicon** without rebuilding `llama.cpp`
  manually for each.
- You ship models through **OCI registries** and want your model
  artifacts to live next to your container images, with the same
  RBAC and signing story.
- You are on **Fedora / RHEL / CoreOS** where Podman is the
  default container engine and the project is a first-class
  package.

**When not to choose.**

- You want **the simplest possible local runner** with no
  container engine. Use `llamafile` (single binary) or Ollama
  (single daemon).
- You want a **chat TUI**, not a runtime. Use
  [`oterm`](../oterm/) on top of Ollama, or
  [`elia`](../elia/) for a multi-provider chat surface.
- You need **closed-weight cloud models**. `ramalama` is
  intentionally local-only; use [`mods`](../mods/),
  [`aichat`](../aichat/), [`llm`](../llm/), or any agent CLI.

## Sample command + expected shape

```
$ ramalama info
Engine:        podman 5.4.1
Image family:  cuda (CUDA 12.4 detected)
Store:         /home/me/.local/share/ramalama
Image:         quay.io/ramalama/cuda:latest

$ ramalama pull ollama://granite3-dense:8b
Pulling granite3-dense:8b ... 4.7 GB done.

$ ramalama serve --port 11434 granite3-dense:8b &
[+] Container ramalama-granite3-dense-8b started; OpenAI API at
    http://localhost:11434/v1

$ curl -s http://localhost:11434/v1/chat/completions \
    -H 'content-type: application/json' \
    -d '{"model":"granite3-dense:8b","messages":[{"role":"user","content":"hi"}]}' \
    | jq -r '.choices[0].message.content'
Hi! How can I help?

# Now point any OpenAI-compatible client at it:
$ OPENAI_API_BASE=http://localhost:11434/v1 OPENAI_API_KEY=local sgpt "explain xargs -P"
```

## Links

- Source: <https://github.com/containers/ramalama>
- Project site: <https://ramalama.ai>
- Container images: <https://quay.io/organization/ramalama>
