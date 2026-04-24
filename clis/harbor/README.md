# harbor

> Snapshot date: 2026-04. Upstream: <https://github.com/av/harbor>
> Binary name: `harbor`

`harbor` is the **local-LLM-stack launcher** of the catalog. It is
not a model client, not an agent, not a chat UI — it is the
Docker-Compose-in-a-trenchcoat that takes a one-line wish like
"give me Open WebUI plus Ollama plus a web-search RAG plus
voice", expressed as `harbor up open-webui ollama searxng
speaches`, and brings up a fully pre-wired stack of containers
with the cross-service URLs, env vars, networks, and volumes
already correctly threaded so you can use it instead of debugging
why service A cannot reach service B.

It earns a catalog slot for being the only entry whose value is
**stack assembly across dozens of independent local-LLM
projects**. Where [`ollama`](../ollama/) /
[`llamafile`](../llamafile/) / [`localai`](../localai/) /
[`ramalama`](../ramalama/) each give you *one* runtime and
[`oterm`](../oterm/) / [`parllama`](../parllama/) give you *one*
front-end, `harbor` gives you the **glue** for the dozens of
combinations.

The intended loop:

1. `curl ... | bash` (or `brew tap av/harbor && brew install
   harbor`) installs the CLI.
2. `harbor up` boots the default profile (Open WebUI + Ollama).
3. `harbor up <service> <service> ...` brings additional services
   up, pre-configured to talk to the existing stack — no manual
   YAML edits, no copy-pasting Docker network names.
4. `harbor down` tears it all down; volumes persist between runs
   under a known prefix so model caches survive.

## 1. Install footprint

- The CLI is **a Bash entrypoint** plus a tree of Compose files,
  service profiles, and helper scripts. Runs anywhere Docker
  Engine + `docker compose` work.
- Install paths:
  - One-liner installer: `curl -fsSL https://get.harbor.sh | bash`
    (drops a `harbor` shim on PATH and clones the service tree
    under `~/.harbor/`).
  - Homebrew tap: `brew tap av/harbor && brew install harbor`.
  - From source: `git clone https://github.com/av/harbor && cd
    harbor && ./harbor.sh link`.
- Hard prereq: **Docker Engine** (or Docker Desktop, OrbStack,
  Colima) with the Compose v2 plugin. Most services are
  CPU-friendly; GPU services need the NVIDIA Container Toolkit
  (or ROCm equivalent) configured at the Docker layer.
- Optional **Harbor App** is a desktop companion (Tauri-based)
  that surfaces logs, model management, and a built-in terminal
  on top of the same Compose stacks. The CLI is the source of
  truth; the App is a viewer.
- Disk footprint scales with services chosen — a "just Open
  WebUI + Ollama" install lands ~3 GB before model weights.

## 2. Repo, version, license

- Repo: <https://github.com/av/harbor>
- Version: **v0.4.10** (2026-04).
- License: Apache-2.0. License file at the repo root:
  [`LICENSE`](https://github.com/av/harbor/blob/main/LICENSE).
- Language: Bash (CLI) + YAML (Compose service catalog) +
  TypeScript (Harbor App).

## 3. Models supported

Harbor itself does not load models — that is delegated to whatever
inference backend you `harbor up`. Supported backends as of
v0.4.x include:

- **Inference backends.** Ollama, llama.cpp, vLLM, TGI,
  LiteLLM, Aphrodite, KoboldCpp, OpenedAI Speech, Faster-Whisper,
  ComfyUI, Stable Diffusion WebUI, Flux, Speaches, Piper, plus
  cloud-provider proxies (MiniMax, Cerebras, Groq, etc.) where
  you supply API keys.
- **Frontends.** Open WebUI, LibreChat, SillyTavern, Lobe Chat,
  HuggingFace Chat UI, Aider, Plandex, Continue, plus the
  Harbor App itself.
- **Glue services.** SearXNG (web RAG), Tika (doc extraction),
  Qdrant / Chroma / Weaviate (vector DBs), Langfuse (LLM
  observability), Boost (Harbor's own routing/transformation
  layer that injects modules like `analogical`, `deaf`, etc.
  into any backend's responses), Hermes (agent runtime), JupyterLab.

The currency is the **service**, not the model — `harbor pull
llama3.1` proxies to whatever backend is up.

## 4. MCP support

Harbor itself does not speak MCP. **Services it brings up may** —
e.g., the included `ros-mcp-server` profile, or any frontend
service (Open WebUI, LibreChat) configured against an MCP-aware
agent. Harbor's job is to start them with the right env vars and
network connectivity; the MCP wiring lives inside the individual
services.

## 5. Sub-agent model

None. There is no agent loop in `harbor` itself. The included
**Hermes** service is an opt-in agent runtime you can `harbor up
hermes` and point at the local stack — but Hermes is the agent,
Harbor is the launcher.

## 6. Telemetry stance

**Off.** No analytics in the CLI or App. Network egress is
exactly: image pulls (Docker Hub, GHCR), model pulls (the
configured backend's registry — Ollama Hub, HuggingFace, etc.),
and any cloud-provider service you opt into. The Compose
networks Harbor creates are local by default; nothing phones
home about which services you launched.

## 7. Token / context strategy

N/A. Harbor does not see prompts or tokens — those flow through
whatever inference backend is active. **Boost** (the optional
routing/transformation module) can intercept and rewrite prompts
in flight (system-prompt injection, response post-processing),
which is the closest thing to a Harbor-level "context strategy",
but it is opt-in per backend.

## 8. Hot keybinds

CLI is flag-and-subcommand. Most useful shapes:

- `harbor up` — start the default profile (Open WebUI + Ollama).
- `harbor up <svc1> <svc2> ...` — additionally start named
  services, pre-wired to the running stack.
- `harbor down` — stop everything; volumes persist.
- `harbor ps` — list running services and their URLs.
- `harbor open <svc>` — open the service's web UI in your browser.
- `harbor logs <svc>` — tail logs for a service.
- `harbor pull <model>` — proxy `ollama pull` (or equivalent for
  the active backend).
- `harbor config <svc>.<key> <value>` — set a service-specific
  config knob without hand-editing YAML.
- `harbor link` / `harbor unlink` — install the wrapper into PATH.
- `harbor dev docs` — regenerate the wiki docs (maintainer-only).

## 9. Killer feature, weakness, when to choose

**Killer feature.** **One-command, pre-wired multi-service local
LLM stacks.** `harbor up open-webui ollama searxng speaches
comfyui` gives you, in one command, a chat UI + a model runtime +
web RAG + voice + image generation, with all the cross-service
URLs already correct. The hand-rolled equivalent is a
several-hundred-line Compose file you maintain forever, plus the
ongoing tax of every service's URL/env-var quirks. Harbor takes
that tax.

**Weakness.**

1. **Docker is a hard requirement.** No Docker, no Harbor. On
   constrained hosts (corp laptops with no Docker license,
   air-gapped servers without an OCI registry mirror), this is
   a deal-breaker.
2. **Bash + Compose YAML is the source of truth.** When something
   breaks (a service version drifted, an env var renamed
   upstream), debugging means reading Compose files and shell
   scripts — not a structured config language. The breadth of
   the service catalog is paid for in install-tax surprises.
3. **Not a runtime, not a frontend, not an agent.** Harbor is
   pure assembly; if you only need *one* service, install it
   directly. `brew install ollama && ollama run llama3.1` is
   shorter than `harbor up`.

**When to choose.**

- **You want a *full* local LLM environment** (chat + RAG +
  voice + maybe image) on your workstation and you do not want
  to maintain the Compose yourself.
- **You are evaluating frontends or backends in pairs** — Open
  WebUI vs LibreChat against Ollama vs vLLM vs llama.cpp.
  `harbor up <combo>` / `harbor down` makes the combinations
  cheap.
- **You want Boost-style prompt transformations** layered on top
  of an arbitrary local backend without modifying the backend.

**When not to choose.**

- **You only need a runtime** → use [`ollama`](../ollama/),
  [`llamafile`](../llamafile/), [`localai`](../localai/), or
  [`ramalama`](../ramalama/) directly.
- **You only need a chat front-end** → use
  [`oterm`](../oterm/) / [`parllama`](../parllama/) /
  [`elia`](../elia/) — no Docker required.
- **You are already on Docker Desktop with hand-rolled Compose
  files you understand** — Harbor's value is the curation; if
  you do not need the curation, the abstraction is in the way.

## 10. Compared to neighbors in the catalog

| Tool | Layer | Brings its own UI? | Brings its own runtime? | Glue (RAG / voice / image / web search) | Install vector |
|------|-------|--------------------|-------------------------|------------------------------------------|----------------|
| harbor | Stack assembler (Compose) | No (pulls in Open WebUI / LibreChat / SillyTavern / Lobe / etc. as services) | No (pulls in Ollama / vLLM / llama.cpp / TGI / etc. as services) | Yes — SearXNG, Tika, Qdrant, ComfyUI, Speaches, Piper as one-liner services | Bash + Docker Compose |
| [ollama](../ollama/) | Runtime | No (HTTP API + REPL) | Yes (built-in `llama.cpp` fork) | No | Single Go binary |
| [llamafile](../llamafile/) | Runtime + weights | Minimal browser UI on `:8080` | Yes (Cosmopolitan binary fuses runtime + weights) | No | Single executable |
| [localai](../localai/) | Multi-modal serving layer | Optional admin UI | Yes (multi-backend) | Partial — bundles chat/embed/STT/TTS/image/rerank under one OpenAI HTTP surface | Single Go binary or Docker |
| [ramalama](../ramalama/) | Containerised runtime | No | Yes (auto-selects `llama.cpp` image per hardware) | No | Python CLI + Podman/Docker |
| [oterm](../oterm/) / [parllama](../parllama/) / [elia](../elia/) | Frontend | Yes (TUI) | No (talks to existing backend) | No | Python |

Decision shortcut:

- "Just give me a model on this laptop in one command":
  [`ollama`](../ollama/) or [`llamafile`](../llamafile/).
- "OpenAI HTTP shape covering chat + embeddings + STT + TTS +
  image": [`localai`](../localai/).
- "Container-native runtime, hardware-aware image":
  [`ramalama`](../ramalama/).
- "I want a chat TUI, not a webapp":
  [`oterm`](../oterm/) / [`parllama`](../parllama/) /
  [`elia`](../elia/).
- "I want **the whole stack** — chat UI + runtime + web RAG +
  voice + image — wired up correctly in one command, and I
  already run Docker": `harbor`.
