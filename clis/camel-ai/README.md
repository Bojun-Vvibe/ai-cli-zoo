# camel-ai

- **Repo:** https://github.com/camel-ai/camel
- **Version:** `v0.2.90` (latest release)
- **License:** Apache-2.0 (`LICENSE`)
- **Language:** Python
- **Install:** `pip install "camel-ai[all]"` then `python -m camel.cli`

## One-line summary

A multi-agent framework whose research thesis is **role-playing** — two (or N)
LLM agents adopt distinct system prompts (e.g. "Python Programmer" ↔ "Stock
Trader") and converse to solve a task, with the conversation transcript
itself as both the deliverable and a corpus for studying agent scaling laws.

## What it does

CAMEL ("Communicative Agents for Mind Exploration of Large-language-model
society") was one of the earliest open-source multi-agent frameworks (Mar
2023) and has since grown into a general-purpose toolkit covering:

- **Society types**: `RolePlaying` (two agents, user ↔ assistant role),
  `Workforce` (manager LLM + worker pool with task decomposition),
  `BabyAGI`-style continuous task lists, hierarchical agents.
- **20+ model backends** via a unified `ModelFactory`: OpenAI, Anthropic,
  Gemini, Mistral, Groq, DeepSeek, Qwen, Zhipu/GLM, Moonshot, NVIDIA NIM,
  SambaNova, Cohere, Together, Reka, vLLM, Ollama, llama.cpp, any
  OpenAI-compatible base URL.
- **Toolkits**: 50+ pre-built tool sets (browser, code-exec via E2B,
  search, GitHub, ArXiv, Notion, Slack, Google APIs, file I/O, RAG,
  HuggingFace, image generation, audio).
- **Data generation**: shipped scripts that spawn N×M role-playing
  conversations to bootstrap fine-tuning datasets — the project's
  original purpose.

CLI entry points (`camel chat`, `camel society`, etc.) are thin; most use
is library-mode driving an `RolePlayingSession`.

```python
from camel.societies import RolePlaying
from camel.types import ModelType, ModelPlatformType
from camel.models import ModelFactory

model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI,
    model_type=ModelType.GPT_4O_MINI,
)

session = RolePlaying(
    assistant_role_name="Python Programmer",
    user_role_name="Stock Trader",
    task_prompt="Develop a trading bot for the stock market",
    assistant_agent_kwargs={"model": model},
    user_agent_kwargs={"model": model},
)

input_msg = session.init_chat()
for _ in range(10):
    a, u = session.step(input_msg)
    print(a.msg.content)
    input_msg = a.msg
```

## When to choose it

- You want to study or generate **multi-agent dialogue corpora** — CAMEL's
  research lineage means data-gen scripts are first-class.
- You want a **single framework that covers role-play, hierarchical
  workforce, and task-list patterns** without picking three libraries.
- You want first-class access to Chinese model providers (Qwen, Zhipu,
  Moonshot, DeepSeek) alongside the usual Western set.

## When NOT to choose it

- You want a typed, production-shaped agent runtime — reach for
  `pydantic-ai`, `openai-agents-python`, or `agno` instead.
- You want a coding agent that edits your repo — CAMEL is a research
  framework, not a code-editing CLI; pair with `aider` / `opencode`.
- You need MCP-native tool mounting — CAMEL has its own toolkit
  abstraction; MCP support exists but is not the central pattern.
