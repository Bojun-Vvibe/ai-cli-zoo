# truss

> Snapshot date: 2026-04. Upstream: <https://github.com/basetenlabs/truss>
> Pinned release: `v0.16.2` (HEAD `c7f32a794b23d9027ba6e2c58a9be7bf43595191`).
> License file: [`LICENSE`](https://github.com/basetenlabs/truss/blob/main/LICENSE)
> (sha `87c8f18a84f8b4ef0f79324c58f705b200d29621`, MIT).

`truss` is a Python CLI that turns a model — a HuggingFace
checkpoint, a `transformers` / `diffusers` script, a vLLM /
SGLang / TensorRT-LLM launch config, or arbitrary Python serving
code — into a containerised inference endpoint, then ships it
either to Baseten's hosted GPU fleet or to a local Docker for
development. The CLI's job is the boring middle: build the
right CUDA base image, pin Python deps, wire a health probe and
an OpenAI-compatible HTTP shim, push the image, and surface logs
and metrics back. It is the catalog's reference for **a model-
serving CLI** that operates one layer below the agent CLIs:
`truss push` is what stands up the endpoint that
[`aider`](../aider/) / [`aichat`](../aichat/) /
[`forge`](../forge/) point `OPENAI_BASE_URL` at.

## 1. Install footprint

- `pip install --upgrade truss` (Python 3.9+) or
  `uvx truss <command>` for one-shot runs without a global
  install.
- Local Docker is required for `truss image build` / `truss
  predict` against a built image; remote `truss push` only needs
  a Baseten API key.
- A `truss` model lives in a directory with `config.yaml` (model
  id, hardware request, Python deps, build commands) plus an
  optional `model/model.py` for custom serving logic. No
  Dockerfile, no Kubernetes YAML.

## 2. Repo, version, license

- Repo: <https://github.com/basetenlabs/truss>
- Latest release: `v0.16.2` (2026-04-22).
- HEAD pinned at this snapshot:
  `c7f32a794b23d9027ba6e2c58a9be7bf43595191`.
- License: MIT. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/basetenlabs/truss/blob/main/LICENSE)
  (sha `87c8f18a84f8b4ef0f79324c58f705b200d29621`).

## 3. What it actually does

Verbs are organised around the model-as-directory abstraction:

```
truss init my-model            # scaffold config.yaml + model/
truss push                     # build container, push to Baseten
truss predict -d '{"prompt":"hi"}'   # call the deployed endpoint
truss image build              # build the Docker image locally
truss image run                # run it locally on :8080
truss watch                    # live-reload model code into the
                               # running container
truss train push               # launch a Baseten training job
truss chains push              # deploy a multi-step Chains pipeline
truss logs                     # tail deployment logs
```

`config.yaml` is the surface area: pick the engine
(`vllm`, `sglang`, `trt-llm`, `transformers`), the hardware
(`A10`, `A100`, `H100`, `H200`, `B200`), the GPU count, the
secret names to mount, and `truss` resolves the rest into a
buildable image. The deployed endpoint is OpenAI-compatible by
default for LLMs (chat / completions / embeddings shapes), so
the agent CLIs in this catalog drop in unchanged.

## 4. MCP support

None. `truss` is a deployment tool, not an agent runtime; the
endpoints it stands up are HTTP / OpenAI-compatible only. MCP
support, if needed, lives in the consumer CLI that calls the
endpoint.

## 5. Sub-agent model

None at the CLI level. `truss chains` does ship a server-side
multi-step composition primitive (one Chain stage calls the
next over RPC inside the Baseten cluster), but it is a serving
topology, not an agent loop — there is no planner / executor
LLM split.

## 6. Telemetry stance

Mixed. The CLI itself does not phone home for usage data; `truss
push` necessarily talks to `app.baseten.co` to upload the build
context, and the deployed model emits operational metrics back
to the Baseten control plane. Local-only flows
(`truss image build`, `truss image run`, `truss predict
--remote-url http://localhost:8080`) need no network beyond the
base-image pulls.

## 7. Token / context strategy

Per-engine, declared in `config.yaml`. For LLM serving via vLLM
the relevant knobs (`max_model_len`, `tensor_parallel_size`,
`gpu_memory_utilization`, KV-cache dtype, speculative decoding)
go under `build.arguments` and are forwarded verbatim to the
engine launch. `truss` does not impose its own context window;
it inherits the model's.

## 8. Hot keybinds

None — `truss` is a non-interactive command-runner. `truss
watch` and `truss logs` are streaming tails (`Ctrl-C` to exit).

## 9. Killer feature, weakness, when to choose

**Killer feature.** **`config.yaml` → production GPU endpoint
in one command, with no Docker / Kubernetes hand-rolled.** The
hard parts of GPU serving — picking the right CUDA base image
for the engine, building TensorRT engines on first boot,
mounting secrets, wiring an OpenAI-compatible shim around a
non-OpenAI model, autoscaling cold-starts — are absorbed into
config the user writes once. The same `truss push` works
against a local Docker for iteration and against a hosted GPU
fleet for production, so the dev → prod gap shrinks to a flag.

**Weakness.** The hosted-deploy path is wired to one vendor
(Baseten); `truss image build` produces a portable Docker image,
but the niceties (`truss watch` live reload, `truss chains`,
managed autoscaling, the deploy UI) only light up against
Baseten. Self-hosters who want a vendor-neutral
package-and-serve story are better off with
[`bentoml`](../bentoml/) (multi-cloud yatai backend) or a
[`vllm`](../vllm/) / [`sglang`](../sglang/) launch wrapped in
their own Helm chart.

**Choose `truss` when** the team has settled on Baseten as the
GPU substrate and wants the shortest path from a HuggingFace id
to a production OpenAI-compatible endpoint that the rest of
this catalog's agent CLIs can consume. **Choose something else
when** vendor neutrality matters
([`bentoml`](../bentoml/)), the workload is local-only
([`ollama`](../ollama/), [`lms`](../lms/),
[`ramalama`](../ramalama/)), or batched-throughput tuning is
the actual problem ([`vllm`](../vllm/), [`sglang`](../sglang/),
[`text-embeddings-inference`](../text-embeddings-inference/)).
