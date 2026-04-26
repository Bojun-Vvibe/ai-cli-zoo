# claudette

- **Repo:** https://github.com/AnswerDotAI/claudette
- **Version:** 0.3.14 (latest release)
- **License:** Apache-2.0 (`LICENSE`)

## What it does

A small, opinionated Python wrapper around Anthropic's Claude API from
the AnswerDotAI / fast.ai folks. Adds stateful `Chat` objects, automatic
tool-use loops, image/PDF inputs, prefill, prompt caching, and a REPL
that's pleasant to drive interactively. Built on `nbdev` so the source
doubles as literate notebooks.

## Install

```sh
pip install claudette
```

## Usage

```python
from claudette import Chat, models

chat = Chat(model=models[1])  # claude-3-5-sonnet
chat("Write a haiku about jq")

# Tool-use loop, no manual scheduling
def get_weather(city: str) -> str: ...
chat = Chat(model=models[1], tools=[get_weather])
chat.toolloop("What's the weather in Reykjavik?")
```

## Why it's interesting

Most "Claude SDK" libraries either match the raw HTTP shape exactly or
hide everything behind agent framework abstractions. Claudette sits in
the middle: the `Chat.toolloop` primitive is a useful 30-line agent loop
with no ceremony, prompt caching is one keyword, and the whole thing is
small enough to read in an afternoon. Good base for one-off scripts that
need a real conversation.
