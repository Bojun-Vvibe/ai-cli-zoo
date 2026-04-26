# art

> Snapshot date: 2026-04-26. Upstream: <https://github.com/OpenPipe/ART>

"**Agent Reinforcement Trainer: train multi-step agents for
real-world tasks using GRPO.**" ART (Agent Reinforcement Trainer)
is OpenPipe's open-source library for fine-tuning open-weight
models on **multi-turn agent rollouts** with GRPO — give your
agent a task + a reward function, let it run, and ART trains the
underlying LLM on its own trajectories so the agent gets
measurably better at the loop it actually executes (browsing,
tool use, code edits, ticket resolution).

## 1. Install footprint

- Library: `pip install openpipe-art` exposes the `art` Python
  package; pair with vLLM / Unsloth for the inference + LoRA
  training backend.
- CLI: `art` entrypoint for managing trainable model snapshots and
  inspecting rollout buffers; trainer loop is driven from a Python
  script (`art.TrainableModel(...).train(rollout_fn, reward_fn)`).
- RULER: built-in **R**elative **U**niversal **L**LM-**E**licited
  **R**ewards mode lets a judge LLM produce per-trajectory rewards
  when no programmatic reward function exists.

## 2. Repo + version + license

- Repo: <https://github.com/OpenPipe/ART>
- Latest release: **v0.5.17**
- License: **Apache-2.0** —
  <https://github.com/OpenPipe/ART/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Stars: ~9.2k

## 3. Models supported

Trainable backbones: Qwen2.5 / Qwen3, Llama 3.x, GPT-OSS, Mistral,
DeepSeek, any HF causal LM that vLLM + Unsloth can serve. Inference
during rollouts: vLLM (default), any OpenAI-compatible endpoint for
the judge / RULER step. Storage: local LoRA checkpoints + S3 for
long-running runs.

## 4. Notable angle

**RL fine-tuning targeted at agent loops, not single-turn chat.**
Where DPO / SFT pipelines flatten an agent run into individual
prompts, ART keeps the **whole multi-step trajectory** as the
optimization unit and applies GRPO so the model is rewarded for
the path that hit the goal, not just the final token. RULER lets
teams skip the months of reward-engineering — the same judge-LLM
pattern that scores eval suites doubles as the reward signal —
and the published case studies (email-search agent, game-playing
agent, codebase-Q&A agent) show open Qwen / Llama checkpoints
catching up to or beating frontier closed models on the trained
task at a fraction of the inference cost.

## 5. Last verified

2026-04-26 via `gh api repos/OpenPipe/ART`.
