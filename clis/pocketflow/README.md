# pocketflow

- **Repo:** https://github.com/The-Pocket/PocketFlow
- **Version:** v0.0.0 (tag dated 2025-03-25; project versions via PyPI thereafter)
- **License:** MIT (`LICENSE` @ SHA `3636748e048698ef7a86a3f7b6143392a7aa5808`)

## What it does

Pocket Flow is a deliberately tiny LLM agent framework — the core fits in
roughly 100 lines of Python — that models any agent as a directed graph of
**Nodes** connected by **Actions**. A `Node` has three lifecycle hooks
(`prep`, `exec`, `post`); the value `post` returns is the action label the
graph follows to the next node. There are no hidden globals, no implicit
prompt scaffolding, and no vendor lock-in: BYO LLM client, BYO tools, BYO
storage.

## Install / Run

```sh
pip install pocketflow
```

```python
from pocketflow import Node, Flow

class Answer(Node):
    def prep(self, shared): return shared["question"]
    def exec(self, q):       return call_llm(q)            # your client
    def post(self, shared, _, ans):
        shared["answer"] = ans
        return "default"

flow = Flow(start=Answer())
flow.run({"question": "Why is the sky blue?"})
```

Branching, retries, batch processing, and async variants (`AsyncNode`,
`AsyncParallelBatchNode`) all reuse the same prep/exec/post contract.

## AI-native angle

The whole pitch is "let agents build agents": because the framework is
small enough for an LLM to hold in one context window, it is well-suited to
code-generation workflows where one agent emits the Pocket Flow graph for
the next agent to execute. Pick it over heavier orchestrators when you
want a substrate you can fully read in an afternoon and a graph
representation that survives translation between Python, TS, and Go ports
maintained in the same org.
