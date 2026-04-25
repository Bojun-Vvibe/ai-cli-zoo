# huggingface-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/huggingface_hub>
> Binary name: `huggingface-cli` (also `hf` shim in newer versions)

`huggingface-cli` is the **model-and-dataset transport layer** of the
catalog. Almost every other entry that runs a local model (`llama.cpp`,
`vllm`, `sglang`, `lmdeploy`, `mlx-lm`, `ollama` via Modelfile imports,
`unsloth`, `axolotl`) ultimately needs weights pulled from the Hugging
Face Hub, and `huggingface-cli` is the canonical Python-shipped CLI for
that pull. It also handles the inverse: `huggingface-cli upload` pushes
fine-tunes back to a repo, with chunked resumable transfer baked in.

The CLI wraps the same `huggingface_hub` Python client every other tool
imports, so authentication (`huggingface-cli login` writes
`~/.cache/huggingface/token`) is shared across `transformers`,
`datasets`, `diffusers`, and any inference server that reads the same
token file. That shared-credential property is the actual reason this
entry belongs in a CLI catalog — it is the **single source of truth for
"am I logged into the Hub"** on a workstation.

## 1. License

Apache-2.0. License file: [`LICENSE`](https://github.com/huggingface/huggingface_hub/blob/main/LICENSE).

## 2. Latest version

`v1.11.0` (released 2026-04-16) — "Semantic Spaces search, Space logs,
and more." Versioning is semver-ish; the `huggingface_hub` Python
package and the `huggingface-cli` binary ship from the same release tag.

## 3. Install

```bash
pip install -U "huggingface_hub[cli]"
huggingface-cli login                              # paste token from https://huggingface.co/settings/tokens
huggingface-cli download meta-llama/Llama-3.1-8B-Instruct \
  --local-dir ./models/llama-3.1-8b
```

The `[cli]` extra pulls in `InquirerPy` for interactive prompts; the
core download/upload functions work without it but the login flow is
nicer with it installed.

## 4. When to choose

- You are running **any local-inference stack** in this catalog and need
  weights on disk. Use `huggingface-cli download` rather than `git lfs
  clone` — it does parallel chunked transfer, resume, and integrity
  checks that plain git-lfs does not.
- You are **publishing fine-tunes** from `axolotl` / `unsloth` /
  `mlx-lm` and want a one-line push: `huggingface-cli upload
  myorg/my-finetune ./out/`.
- You want **shared auth** across every Python AI tool on the box —
  log in once, every `from huggingface_hub import ...` call is
  authenticated.
- Avoid if your workflow is **purely API-hosted** (OpenAI, Anthropic,
  Groq, Replicate) and you never touch local weights — you do not need
  this.
