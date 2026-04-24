# smolagents

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/smolagents>
> Latest release: **v1.24.0** (2026-01-16). License: **Apache-2.0** ([`LICENSE`](https://github.com/huggingface/smolagents/blob/main/LICENSE)).

Hugging Face's "small" agent framework whose defining choice is **the agent
writes Python code, then a sandbox executes it** — instead of structured
tool-call JSON. Ships a `smolagent` CLI that takes a prompt, picks a model
+ tools, and runs the code-action loop until done.

## 1. Install footprint

- `pip install smolagents` (core ~5 MB) or `pip install 'smolagents[all]'`
  (pulls `litellm`, `e2b`, `docker`, `selenium`, `transformers`, MCP client,
  vision, audio — ~600 MB).
- Python 3.10+. `smolagent` and `webagent` console-scripts land on PATH.
- Optional sandboxes: local Python (default, **unsandboxed — be aware**),
  E2B remote sandbox, Docker, or WebAssembly via `wasmer`.

## 2. License

Apache-2.0 (file: `LICENSE`, SPDX `Apache-2.0`). Permissive, no patent
landmines, safe to embed.

## 3. Models supported

Anything with an `InferenceClient` adapter:

- Hugging Face Inference Providers (`HfApiModel` — Together, Replicate,
  Sambanova, Fireworks, Novita, Hyperbolic, Cerebras routed via HF).
- LiteLLM (`LiteLLMModel`) — OpenAI, Anthropic, Gemini, Bedrock, Ollama,
  vLLM, OpenRouter, ~100 more.
- Local `transformers` (`TransformersModel`) — runs the model in-process.
- Azure OpenAI, OpenAI-compatible HTTP (`OpenAIServerModel`).
- MLX on Apple Silicon (`MLXModel`).

The framework was specifically designed to make small open-weight models
(Qwen2.5-Coder-32B, DeepSeek-V2.5, Llama-3.3-70B) viable agent backbones,
so the "code-as-action" choice is partly a bet that small models write
Python better than they emit valid tool-call JSON.

## 4. MCP support

Yes (client). `smolagents.mcp_client.MCPClient` mounts an MCP server's
tools as smolagents `Tool` objects — same shape as `tool` decorators —
and the agent can call them inside its generated Python (`mcp_tool(...)`
becomes a function call in the code action). No first-party MCP server
mode.

## 5. Sub-agent model

`ManagedAgent` wraps any agent as a callable tool, so a top-level
`CodeAgent` can delegate to a sub-`CodeAgent` or `ToolCallingAgent`
specialised for, e.g., web search or vision. Composition is recursive
and explicit (you wire it in Python), not auto-spawned.

## 6. Telemetry stance

**Off.** No analytics in the package itself. Optional opt-in OpenTelemetry
hook (`smolagents.utils.tracing`) lets you push traces to Arize Phoenix,
Langfuse, or any OTel collector — disabled until you call it.

## 7. Prompt-cache strategy

Whatever the underlying provider does. No smolagents-side response cache.
The agent's running Python state *is* the cache for intermediate results
within a single task — variables defined in step N are visible in step
N+1's code action.

## 8. CLI surface

```
smolagent "prompt..."                        # CodeAgent over default model + tools
smolagent "prompt..." --model-type LiteLLMModel \
  --model-id anthropic/claude-sonnet-4 \
  --tools web_search,visit_webpage \
  --imports requests,bs4
webagent "prompt..."                         # Selenium-driven web agent
```

Plus a Gradio `GradioUI(agent).launch()` if you want a browser chat over
the same agent.

## 9. Strengths

- **Code-as-action is genuinely different.** A `CodeAgent` step emits
  Python like `results = web_search("…"); summary = llm(results[:3])`
  — control flow, loops, list comprehensions, error handling all happen
  in one model call instead of a 6-turn JSON tool-call dance. Cuts token
  usage and latency on multi-step tasks measurably.
- **Smallest agent framework in the catalog.** Core is ~1k LoC, the
  abstractions (`Agent`, `Tool`, `Model`) fit in your head in 30 minutes.
  You can read the whole `CodeAgent.run()` loop top-to-bottom.
- **Sandbox menu is honest about the tradeoff.** Default is local Python
  (fast, unsafe), but `executor_type="e2b"` / `"docker"` / `"wasm"` are
  one flag away and the docs are loud about which tasks need which.

## 10. Weaknesses / when not to use

- **Local-Python default is a footgun.** The agent will `os.system(...)`
  or `shutil.rmtree(...)` if you ask it to and you forgot to set a
  sandbox. Anyone running smolagents in CI without `executor_type="e2b"`
  or `"docker"` is effectively giving the LLM shell on the host. Use
  `aider` / `cline` if you want approve-each-step trust UX out of the box.
- **Code-as-action is wrong for narrow well-typed APIs.** If the task is
  "call these three GraphQL mutations in the right order", structured
  tool-call agents (claude-code, codex, opencode) waste fewer tokens and
  fail more legibly than smolagents emitting Python that wraps the same
  three calls.

## 11. Comparison vs nearest neighbor in zoo

The closest catalog entry is **[`open-interpreter`](../open-interpreter/)** —
both run model-generated code in a local interpreter loop. The split:
`open-interpreter` is a **REPL product** (interactive chat where the model
writes-and-runs Python/shell turn by turn, optimised for "I'm pair-programming
with the model"). `smolagents` is a **framework + thin CLI** (you build an
`Agent` with explicit tools and sandbox, the CLI is a convenience entry; the
real value is embedding `CodeAgent` in your own Python service). Pick
`open-interpreter` for ad-hoc local automation at the prompt. Pick
`smolagents` when you want the same code-as-action pattern as a library you
can wire MCP tools, sub-agents, and a real sandbox into.
