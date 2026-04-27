# aibrix

> Snapshot date: 2026-04. Upstream: <https://github.com/vllm-project/aibrix>
> Pinned release: `v0.6.0` (HEAD `da77b657cad4e5be48669cee33ff8e90d37cb07a`).
> License file: [`LICENSE`](https://github.com/vllm-project/aibrix/blob/main/LICENSE)
> (sha `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`, Apache-2.0).

`aibrix` is the vLLM-project's Kubernetes control plane for
serving LLMs at fleet scale. It is what sits *above*
[`vllm`](../vllm/) / [`sglang`](../sglang/) / [`lmdeploy`](../lmdeploy/)
once a single inference server is no longer enough: an LLM-aware
gateway, an autoscaler that reads engine-level metrics
(running vs. waiting requests, KV-cache utilisation) instead of
generic CPU, a distributed prefix-cache router that steers
prompts to the replica most likely to already hold the cache,
and a LoRA-adapter manager that hot-swaps adapters into a
shared base model without re-deploying. It is the catalog's
reference for **"vLLM, but production-grade across N nodes"** —
sibling to managed offerings, but self-hosted and open.

## 1. Install footprint

- `helm repo add aibrix https://aibrix.github.io/aibrix && helm install aibrix aibrix/aibrix` on a Kubernetes 1.27+ cluster with GPU nodes (NVIDIA / AMD).
- Brings up: gateway (Envoy + custom filters), controller-manager (CRDs: `RayClusterFleet`, `ModelAdapter`, `StormService`, `RoleSet`), autoscaler (`PodAutoscaler` CRD with `kpa` / `apa` / `hpa` modes), distributed KV-cache layer.
- `kubectl apply -f https://raw.githubusercontent.com/vllm-project/aibrix/v0.6.0/samples/quickstart/model.yaml` deploys a single-model serving stack as a smoke test.
- The user-facing surface is the OpenAI-compatible `/v1/chat/completions` endpoint exposed by the gateway service; clients ([`llm`](../llm/), [`aichat`](../aichat/), [`litellm`](../litellm/), any OpenAI SDK) point `OPENAI_BASE_URL` at it.

## 2. Repo, version, license

- Repo: <https://github.com/vllm-project/aibrix>
- Latest release: `v0.6.0` (2026-03-03).
- HEAD pinned at this snapshot:
  `da77b657cad4e5be48669cee33ff8e90d37cb07a`.
- License: Apache-2.0. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/vllm-project/aibrix/blob/main/LICENSE)
  (sha `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`).

## 3. Why it is in the catalog

Single-replica vLLM is well-covered by [`vllm`](../vllm/) and
its serving cousins. What was missing was the layer that
turns it into a fleet:

- **LLM-aware autoscaling**: `PodAutoscaler` reads `vllm:num_requests_running` and KV-cache pressure from the `/metrics` endpoint of each engine, not pod CPU — so scale-out fires before TTFT collapses, not after.
- **Distributed prefix-cache routing**: the gateway hashes prompt prefixes and steers to the replica that already has those tokens in KV — a single shared cache plane across the fleet, not one cache per pod.
- **LoRA-as-a-resource**: `ModelAdapter` CRDs let many tenants share one base model (Llama-3.1-70B, Qwen2.5-72B, DeepSeek-V3) while each loads their own adapter on demand — the production version of what [`peft`](../peft/) / [`lorax`](../lorax/) prototype on a single box.
- **Heterogeneous GPU scheduling**: mixes A100 / H100 / H200 / MI300 nodes with cost-aware placement, instead of treating them as one pool.
- **Stateful inference primitives**: `RayClusterFleet` + `StormService` for disaggregated prefill / decode and multi-host inference of 405B-class models — the patterns this catalog otherwise only sees in vendor blog posts.

## 4. Example invocation

```bash
# 1. Install onto an existing GPU-enabled cluster.
helm repo add aibrix https://aibrix.github.io/aibrix
helm install aibrix aibrix/aibrix --namespace aibrix-system --create-namespace

# 2. Deploy a base model + a LoRA adapter on top of it.
kubectl apply -f - <<'YAML'
apiVersion: orchestration.aibrix.ai/v1alpha1
kind: ModelAdapter
metadata:
  name: code-tutor-lora
spec:
  baseModel: meta-llama/Llama-3.1-8B-Instruct
  artifactURL: huggingface://my-org/code-tutor-lora
  replicas: 2
YAML

# 3. Hit the gateway with a stock OpenAI client.
curl http://aibrix-gateway/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"code-tutor-lora","messages":[{"role":"user","content":"explain monads"}]}'
```

## 5. Where it sits in the catalog

- Above the engine layer ([`vllm`](../vllm/), [`sglang`](../sglang/), [`lmdeploy`](../lmdeploy/), [`text-generation-inference`](../text-generation-inference/)) — aibrix talks to engines via their OpenAI-compatible HTTP surface.
- Adjacent to [`bentoml`](../bentoml/) / [`truss`](../truss/) / [`litserve`](../litserve/) (single-service packaging) but at a different altitude: aibrix is multi-replica, multi-tenant, multi-adapter on Kubernetes, not "wrap one model in an HTTP server."
- Complementary to [`litellm`](../litellm/) on the client side: litellm fans clients out across providers; aibrix fans one cluster out across replicas behind a single OpenAI endpoint.
- Sibling control plane to [`skypilot`](../skypilot/), but skypilot is cluster *provisioning*, while aibrix is the *serving runtime* on top of a cluster you already have.
