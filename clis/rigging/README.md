# rigging

- **Repo:** https://github.com/dreadnode/rigging
- **Version:** v3.3.2 (latest release)
- **License:** MIT (`LICENSE` @ SHA `9609ed3e01022c198d6874e59e64e5c1ae655697`)

## What it does

A lightweight Python framework for building structured LLM interactions. It
treats prompts as composable, typed pipelines: you define Pydantic-style
models for inputs and outputs, chain generators with `.then()` / `.map()`,
and get back validated objects instead of raw strings.

## Install / Run

```sh
pip install rigging
```

```python
import rigging as rg
chat = await rg.get_generator("gpt-4o-mini").chat("Summarize: ...").run()
print(chat.last.content)
```

## AI-native angle

Rigging is built around the assumption that you'll be calling models a lot
and want them to behave like typed functions. The pipeline DSL, retry/parse
loop, and tool-calling primitives are first-class — not afterthoughts —
which makes it a clean fit for agentic workflows where you compose many
small LLM steps rather than one big chat.
