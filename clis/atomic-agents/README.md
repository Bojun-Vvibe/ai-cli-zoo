# atomic-agents

- **Repo:** https://github.com/BrainBlend-AI/atomic-agents
- **Version:** v2.7.5 (latest release)
- **License:** MIT (`LICENSE` @ SHA `37644855ea3b283c4e4c0e622bd935fdc76b7886`)

## What it does

A lightweight, modular framework for building AI agents from small, swappable
parts. Instead of one monolithic agent class, you compose `BaseAgent`,
schemas (input/output Pydantic models), context providers, and tools. Every
piece is meant to be replaceable, testable, and explicit — closer to "Lego
blocks for agents" than a heavy orchestration runtime.

## Install / Run

```sh
pip install atomic-agents
```

```python
from atomic_agents.agents.base_agent import BaseAgent, BaseAgentConfig
from atomic_agents.lib.components.system_prompt_generator import SystemPromptGenerator
# define input/output schemas, plug into BaseAgent, .run(input)
```

A CLI scaffold (`atomic`) is shipped as part of the project for spinning up
new agents quickly.

## AI-native angle

The selling point is determinism and inspectability: typed I/O on every
agent boundary, dependency-injected context providers, and no hidden global
state. Good fit when you want LLM-backed components that behave more like
ordinary functions you can unit-test, rather than opaque chains.
