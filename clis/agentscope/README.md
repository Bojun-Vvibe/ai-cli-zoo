# agentscope

> Snapshot date: 2026-04. Upstream: <https://github.com/agentscope-ai/agentscope>

"**Build and run agents you can see, understand and trust.**"
AgentScope is a multi-agent framework that pairs a typed
message-passing core (`Agent` ↔ `Msg` ↔ `Pipeline`) with a
first-class **AgentScope Studio** — a local web UI that
visualises every agent's inbox, every tool call, every model
prompt, and the inter-agent message graph as it runs. The
framework exists because the authors' bet is that multi-agent
systems become debuggable only when their message flow is
observable in real time, not reconstructed from logs after the
fact. The CLI ships the Studio server, project scaffolds, a
workflow runner, and adapters into the wider model + tool
ecosystem.

## 1. Install footprint

- `pip install agentscope` — pulls pydantic, httpx, FastAPI,
  uvicorn, sqlalchemy; pure Python, ~10 MB.
- Optional extras: `pip install 'agentscope[full]'` to add the
  Studio frontend, distributed runtime (`grpcio`), and the
  service toolkit (web search, code execution, file ops, RAG).
- `as_studio` boots the Studio web UI at `http://localhost:5000`;
  `as_workflow` runs a JSON workflow definition headlessly;
  `as_gradio` exposes any agent as a Gradio chat surface.
- Python ≥ 3.10. Mac / Linux / Windows; the distributed runtime
  uses gRPC and runs across multiple hosts when an agent society
  outgrows a single process.

## 2. Repo + version + license

- Repo: <https://github.com/agentscope-ai/agentscope>
- Latest release: **v1.0.19** (2026-04-20)
- License: **Apache-2.0** —
  <https://github.com/agentscope-ai/agentscope/blob/main/LICENSE>
- HEAD SHA: `eb7678e1cb958125c852027c1a91480a03d7f870`
- Default branch: `main`
- Language: Python (+ TypeScript Studio frontend)

## 3. Models supported

OpenAI (Responses + Chat), Anthropic Claude (Sonnet / Opus /
Haiku), Gemini (AI Studio + Vertex), DashScope (Qwen family,
including Qwen-VL and Qwen-Omni multi-modal), ZhipuAI (GLM-4 /
GLM-4V), Yi, DeepSeek (Chat / Coder / R1), Ollama, vLLM, LiteLLM
proxy, any OpenAI-compatible base URL via `OpenAIChatModel`.
Vision and audio modalities flow through the same `Msg` type via
typed `ImageBlock` / `AudioBlock` / `VideoBlock` content blocks,
so a vision-capable model can be swapped in without rewriting
the agent loop.

## 4. Simple usage

```python
import agentscope as ag
from agentscope.agent import ReActAgent
from agentscope.model import DashScopeChatModel
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit, execute_python_code

ag.init(project="hello-as", studio_url="http://localhost:5000")

toolkit = Toolkit()
toolkit.register_tool_function(execute_python_code)

agent = ReActAgent(
    name="Friday",
    sys_prompt="You are a helpful coding assistant.",
    model=DashScopeChatModel(model_name="qwen-max", api_key="..."),
    formatter=DashScopeChatFormatter(),
    memory=InMemoryMemory(),
    toolkit=toolkit,
)

import asyncio
asyncio.run(agent(ag.message.Msg(
    name="user",
    role="user",
    content="Compute the 30th Fibonacci number with Python and return both the number and the script.",
)))
```

```bash
as_studio &           # boots Studio UI on :5000 → live trace view
python hello_as.py    # the run streams into the UI in real time
```

## 5. Why it's interesting

- **Studio is the differentiator** — every `Msg` exchanged
  between agents, every tool call, every model prompt, every
  state transition streams to the local web UI as it happens;
  multi-agent runs that would be opaque in stdout become a
  navigable timeline + graph + chat-per-agent view, with
  per-step replay.
- **Typed message blocks across modalities** — `Msg(content=
  [TextBlock(...), ImageBlock(url=...), AudioBlock(...)])` is
  the same shape across vision / audio / text, so a multi-modal
  pipeline is the same `Pipeline` API as a text pipeline.
- **Distributed runtime is built in** — `RpcAgent` wraps any
  agent as a gRPC service; an agent society can fan out across
  hosts (one big LLM serves judges on a GPU box, lightweight
  workers run on CPU laptops) without rewriting the orchestrator.
- **Workflow JSON ↔ Studio drag-drop ↔ Python code triangle**
  — a workflow drafted in the Studio canvas exports to JSON,
  runs via `as_workflow`, and the same JSON can be regenerated
  from a Python `Pipeline`; PMs and engineers can meet in the
  middle without forking the source of truth.
- **Hooks at every lifecycle point** (`pre_reply` / `post_reply`
  / `pre_observe` / `post_observe`) — guardrails, telemetry,
  redaction, rate limiting attach as decorators on the agent
  class instead of as a separate middleware service.

## 6. Caveats

- Studio is the value proposition; running headless without it
  loses the strongest reason to pick AgentScope over a thinner
  framework like [`pydantic-ai`](../pydantic-ai/) or
  [`smolagents`](../smolagents/).
- The model + formatter pairing is explicit (DashScope ↔
  DashScopeChatFormatter, OpenAI ↔ OpenAIChatFormatter) — model
  swaps need a formatter swap, which is a learning-curve speed
  bump versus frameworks that hide tool-call dialect behind a
  single `model=` arg.
- Distributed runtime uses gRPC + asyncio under the hood; debug
  scenarios where one remote agent hangs are still less
  ergonomic than a single-process trace.
- Heaviest English-vs-Chinese documentation gap of the
  multi-agent frameworks in the catalog — the Chinese docs are
  often the reference; English docs catch up but lag the latest
  Studio features by a release or two.
