# qwen-agent

- **Repo:** https://github.com/QwenLM/Qwen-Agent
- **Version:** v0.0.26
- **License:** Apache-2.0 (`LICENSE`)
- **Language:** Python
- **Install:** `pip install -U "qwen-agent[gui,rag,code_interpreter,mcp]"`

## One-line summary

Alibaba's official agent framework + CLI for the Qwen model family, with
function-calling, RAG, code interpreter, and MCP tool support out of the box.

## What it does

Qwen-Agent is the upstream-blessed way to build agents on top of Qwen
(via DashScope or any OpenAI-compatible endpoint serving Qwen weights).
It bundles the agent loop with first-party support for:

- **Function calling / tool use** with auto-generated JSON schemas
- **RAG** over long documents (the framework's claim-to-fame is robust
  long-context retrieval that exploits Qwen's native long context)
- **Code interpreter** sandbox for data analysis / plotting
- **MCP (Model Context Protocol)** clients so any MCP server plugs in
- A small **Gradio GUI** and a CLI loop for quick experiments

```python
from qwen_agent.agents import Assistant
bot = Assistant(llm={'model': 'qwen-max'},
                function_list=['code_interpreter'])
for r in bot.run([{'role': 'user', 'content': 'Plot sin(x) for x in 0..2pi'}]):
    print(r)
```

## When to choose it

- You're already on Qwen (DashScope, self-hosted vLLM, or Ollama) and want
  the reference agent harness rather than wiring tool-calling yourself.
- You need **long-context RAG** and want a battle-tested implementation
  tuned for Qwen's tokenizer and context window.
- You want a permissive Apache-2.0 license for downstream products.

## When NOT to choose it

- You're model-agnostic and want a richer multi-provider abstraction —
  prefer `litellm` + a generic agent framework (`pydantic-ai`,
  `mcp-agent`, `crewai`).
- You want a polished interactive coding CLI like `aider` or `claude-code`
  — Qwen-Agent is a framework first, CLI second.
- You can't tolerate Chinese-language docs in the long tail of examples;
  English coverage is good but not exhaustive.
