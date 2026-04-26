# comfyui

> Snapshot date: 2026-04. Upstream: <https://github.com/comfyanonymous/ComfyUI>
> (the GitHub org now redirects to `Comfy-Org/ComfyUI`).

The reference **node-graph runtime for diffusion models** — Stable
Diffusion (1.5, SDXL, SD3, SD3.5), Flux, HunyuanVideo, LTX-Video, Wan,
HiDream, and the long tail of open image / video models. Strictly
out-of-scope for "AI coding CLIs" in the narrowest sense, but included
in this catalog because (a) the **CLI / headless surface** is now
first-class (`python main.py --listen --api-only --workflow-file ...`),
(b) agent frameworks like [`smolagents`](../smolagents/),
[`crewai`](../crewai/), and [`pydantic-ai`](../pydantic-ai/) routinely
drive ComfyUI workflows as a tool when the agent needs to generate /
edit images, and (c) the workflow-as-JSON-graph artefact is what
LLM-based "image agents" emit and execute.

## 1. Install footprint

- `git clone https://github.com/comfyanonymous/ComfyUI && pip install
  -r requirements.txt && python main.py` — that is the supported
  install. Python 3.11+, a working CUDA / ROCm / MPS / DirectML
  backend, and a model checkpoint dropped in `models/checkpoints/`.
- The `comfy-cli` wrapper (`pip install comfy-cli`) provides a
  one-shot installer + node manager + workflow runner, but it is a
  separate package and not the canonical entry.
- CPU-only works for tiny test runs; realistic use is a discrete GPU
  with ≥8 GB VRAM (much more for video models).
- Web UI on `http://localhost:8188`; the same process exposes a JSON
  API (`/prompt`, `/queue`, `/history`) used by every external client.

## 2. Repo, version, license

- Repo: <https://github.com/comfyanonymous/ComfyUI>
- Version checked: **`v0.19.3`** (released 2026-04-17)
- License: **GPL-3.0**. License file:
  <https://github.com/Comfy-Org/ComfyUI/blob/master/LICENSE>
- Default branch: `master`
- The custom-node ecosystem (`custom_nodes/`) is the operational
  surface — the core repo is intentionally small; almost every real
  workflow pulls in 5–30 community nodes via ComfyUI-Manager.

## 3. Models supported

- Image: SD 1.5, SD 2.x, SDXL, SD3 / SD3.5, Flux (.1 / Schnell / Pro
  variants), Pixart, Stable Cascade, Kolors, Lumina, AuraFlow, HiDream,
  any community fine-tune in `safetensors`.
- Video: HunyuanVideo, LTX-Video, Wan, CogVideoX, AnimateDiff
  derivatives, Mochi.
- Audio: Stable Audio Open variants via custom nodes.
- LLM: not natively a generation target — but workflows can call out
  to local Ollama / llama.cpp endpoints via custom nodes for
  prompt-rewriting / agentic loops inside the graph.

## 4. MCP support

Not first-party. Several community MCP servers wrap the ComfyUI HTTP
API to expose `submit_workflow` / `get_image` / `list_models` as MCP
tools — usable from [`opencode`](../opencode/) /
[`claude-code`](../claude-code/) — but none ship in the core repo.

## 5. Sub-agent model

None inherent — ComfyUI is a deterministic dataflow runtime, not an
agent. The interesting pattern is *external*: an LLM agent emits a
workflow JSON (the same format Save Workflow produces), POSTs it to
`/prompt`, and polls `/history/<prompt_id>` for the result. Several
agent frameworks ship a `ComfyUITool` exactly for this — the agent
designs the graph, ComfyUI executes it.

## 6. Telemetry stance

Off. The core process makes no outbound calls beyond model downloads
you initiate. The custom-node ecosystem is a different story —
individual nodes can do whatever they want (and historically *have*,
including a few well-publicised malicious nodes); ComfyUI-Manager now
ships a security-level toggle that gates installs by trust tier.
Treat any custom node as untrusted code.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Workflow as a portable JSON artefact, executable
headlessly behind an HTTP API.** A user (or an LLM) designs a graph in
the web UI, exports `workflow.json`, and any other ComfyUI install can
re-run it bit-exactly given the same model files. That makes ComfyUI
the de-facto **image-generation execution substrate** for LLM agents:
the agent doesn't generate pixels, it generates a workflow JSON and
delegates execution. The community node ecosystem covers every model,
every sampler, every controlnet, every upscaler — usually within days
of a paper drop.

**Weakness.** The custom-node ecosystem is also the supply-chain
attack surface. Workflows are not portable across installs unless the
exact custom nodes + model files are present (resolving "missing
nodes" is its own minor career). The GPL-3.0 license is fine for
internal tooling but not for shipping ComfyUI inside a closed-source
commercial product without legal review. The web UI's spatial-graph
metaphor is powerful but not learnable in 10 minutes; AUTOMATIC1111's
slider-driven UI is friendlier for casual use.

**When to choose.**
- You want **maximum control** over the diffusion pipeline (custom
  samplers, multi-stage refinement, region prompting, controlnets,
  IP-adapter, LoRA stacks, video).
- You want to **drive image / video generation from an LLM agent** —
  the JSON-workflow + HTTP API pattern is the cleanest available.
- You need **headless / batch** inference on a GPU box and a
  reproducible artefact (`workflow.json` + checkpoints) per output.

**When to skip.**
- You only want "type a prompt, get an image" with sensible defaults
  → AUTOMATIC1111 / Forge / Fooocus / SD.Next.
- You need a closed-source-friendly licence → ComfyUI's GPL-3.0
  rules out direct embedding; consider commercial APIs instead.
- Your workload is purely text / code — ComfyUI is not in scope, see
  the rest of this catalog.

## 8. Compared to neighbors

| Tool | Shape | Modality | Agent driver |
|------|-------|----------|--------------|
| comfyui | Node-graph runtime + HTTP API | Image / video diffusion | External LLM emits workflow JSON |
| [llama.cpp](../llama.cpp/) | C++ inference engine | Text (LLM) | Direct |
| [vllm](../vllm/) | Serving framework | Text (LLM) | Direct |
| [replicate-cli](../replicate-cli/) | Hosted model runner | Any (image / text / audio / video) | API call |
