# kit (KitOps)

> Snapshot date: 2026-04. Upstream: <https://github.com/kitops-ml/kitops>

"**Standards-based packaging & versioning for AI/ML projects.**"
KitOps is a CNCF open-source CLI that packages everything an AI/ML
project needs — model weights, datasets, code, prompts, agent specs,
MCP server bundles, configs — into a versioned, signable, tamper-evident
**ModelKit** stored as an OCI artifact in any standard container
registry. The `kit` binary is the entire user surface: `kit init`,
`kit pack`, `kit push`, `kit pull`, `kit unpack`, `kit list`, plus
`kit import` for HuggingFace and `kit dev` for a local serving loop.

## 1. Install footprint

- macOS: `brew tap kitops-ml/kitops && brew install kitops`
  (also `winget install KitOps.kit` on Windows; `.deb` / `.rpm` /
  static tarball for Linux from the releases page).
- Single static Go binary (~30 MB, no runtime deps; one process,
  no daemon).
- Build from source: `go install github.com/kitops-ml/kitops/cmd/kit@latest`
  (Go ≥ 1.22).
- A working `kit pack` only needs a project directory plus a
  `Kitfile` (YAML manifest declaring `model`, `datasets`, `code`,
  `docs`, `metadata`); `kit init .` autogenerates one from the
  directory layout.
- No mandatory cloud account: `kit pack` and `kit unpack` work fully
  offline; `kit push` / `kit pull` need credentials for whichever OCI
  registry you target (Docker Hub, GHCR, ACR, ECR, Artifactory,
  Harbor, Quay, Jozu Hub, or a self-hosted Distribution registry).

## 2. Repo + version + license

- Repo: <https://github.com/kitops-ml/kitops>
- Latest release: **v1.13.0** (2026-04-20)
- License: **Apache-2.0** —
  <https://github.com/kitops-ml/kitops/blob/main/LICENSE>
- Default branch: `main`
- Language: Go
- Governance: CNCF Sandbox project; reference implementation of the
  CNCF **ModelPack** specification (the `kit` CLI reads and writes
  both ModelKit and ModelPack formats transparently).

## 3. Models supported

None directly — KitOps is the *packaging* layer, not a model client.
Inside a ModelKit you can ship any model artifact format you can put
on disk: GGUF (llama.cpp / Ollama), Safetensors / PyTorch `.bin` /
`.pth`, ONNX, TensorFlow SavedModel, Keras `.h5`, scikit-learn
pickles, JAX / Flax checkpoints, MLflow / DVC payloads, and bundled
adapters (LoRA / QLoRA). `kit import <hf-org>/<hf-model>` pulls a
HuggingFace repo into a ModelKit in one command. `kit dev` starts a
local OpenAI-compatible serving loop for the packed model so you can
sanity-check before pushing.

## 4. MCP support

Indirect, via packaging. KitOps does not implement MCP itself, but
ModelKits are the recommended distribution shape for **MCP server
bundles** — pack the MCP server binary, its config, and its
prerequisites into one signed, content-addressable artifact and pull
it into the agent host the same way you pull a container image. There
is no `kit mcp` subcommand; this is a usage pattern, not a feature.

## 5. Sub-agent model

None — KitOps is not an agent. It is the *substrate* an agent
runtime (e.g. [`opencode`](../opencode/), [`goose`](../goose/),
[`agno`](../agno/), [`gpt-engineer`](../gpt-engineer/)) can pull a
versioned model + agent spec + tools bundle from at startup, the way
a microservice pulls its container image.

## 6. Telemetry stance

Off. No analytics in the OSS codebase. Egress is exactly the OCI
registries you `push` / `pull` against, plus HuggingFace on `kit
import`, plus your local `cosign` keys / KMS endpoint if you sign
ModelKits. Project conduct is governed by the CNCF, which mandates
no-telemetry-by-default for sandbox tools.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **One immutable, signable, content-addressable
bundle for the *entire* AI/ML thing — weights + datasets + code +
prompts + agent specs + MCP servers — stored in the OCI registry you
already operate.** Where every other catalog entry treats "the model"
and "the agent harness" and "the dataset" as separate concerns
(downloaded ad hoc from HF / GitHub / S3 at runtime, with no joint
version), KitOps gives you a single SHA-256-addressed artifact with a
declarative `Kitfile`, full Cosign / Sigstore signing, layered storage
that diff-shares unchanged weights across versions, and AI SBOM
generation — so a ModelKit you push from your laptop is byte-identical
to the one production pulls, and the pipeline can verify the signature
before letting it run. CNCF governance + ModelPack interop means it is
the closest thing the catalog has to a vendor-neutral standard.

**Weakness.** **Not a model client and not an agent** — `kit` does
not chat, does not edit code, does not execute tools, does not have
an MCP surface of its own. Adopting it forces you to declare a
`Kitfile` per project and operate an OCI registry (or pay for a hosted
one). The value only shows up at the team / org scale where "which
exact model + dataset + prompt did production run last week" is a real
question; for a single-developer "I just want Claude to edit my
repo" workflow this layer is overhead. Tooling around dev-loop
ergonomics (live-reload during training, partial layer pull-ahead) is
still maturing relative to the OCI ecosystem for application
containers.

**When to choose.** You are operating AI/ML at a team or
organisation scale and need **one versioned, auditable, signable
artifact** that bundles model weights, datasets, prompts, agent
specs, and MCP server payloads, deployable through your existing
container registry + CI/CD + supply-chain controls (Cosign, SBOMs,
attestation gates). You want a vendor-neutral interchange format
(ModelPack / OCI) instead of pinning to one cloud's model registry.
Pair with [`goose`](../goose/) / [`opencode`](../opencode/) /
[`agno`](../agno/) as the agent runtime that pulls the ModelKit at
startup. Skip if you are a solo developer chatting with a hosted LLM
— the manifest + registry overhead is bigger than the win.
