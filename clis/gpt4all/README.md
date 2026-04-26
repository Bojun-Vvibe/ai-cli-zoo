# gpt4all

> Snapshot date: 2026-04. Upstream: <https://github.com/nomic-ai/gpt4all>
> Version pinned at write-time: **v3.10.0**
> License: **MIT** (file: [`LICENSE.txt`](https://github.com/nomic-ai/gpt4all/blob/main/LICENSE.txt))

`gpt4all` is Nomic's open-source stack for **running local LLMs on any
device**. The repo ships three things: a desktop chat app, a C++ backend
(`gpt4all-backend`) that wraps `llama.cpp` with extra format support, and
a Python binding that exposes a small CLI/REPL surface. This entry is
about the Python CLI and the underlying model registry, not the GUI.

It is the right tool when you want a **batteries-included local model
runner with a curated model registry** and don't want to manage `gguf`
URLs by hand.

## 1. Install

```bash
# Python binding (CLI-usable)
pip install gpt4all

# Or install the desktop app via the official installer:
# https://gpt4all.io
```

The Python package downloads model files into
`~/.cache/gpt4all/` on first use. Models are gguf, plus a curated
`models.json` that ships with the package.

## 2. Simple invocation

```python
# A 3-line REPL in Python (the closest thing to a CLI the package ships)
from gpt4all import GPT4All
model = GPT4All("Meta-Llama-3-8B-Instruct.Q4_0.gguf")
with model.chat_session():
    print(model.generate("Summarise rsync in one sentence.", max_tokens=128))
```

```bash
# Or one-shot from the shell
python -c "from gpt4all import GPT4All; m=GPT4All('Meta-Llama-3-8B-Instruct.Q4_0.gguf'); print(m.generate('hello'))"
```

## 3. Why it lives in this catalog

- It is one of the lowest-friction ways to get a working local model on
  a laptop: no manual `gguf` download, no separate server process, no
  GPU required.
- The curated registry at `gpt4all/models.json` is a useful reference
  in its own right — it tracks which open weights are quantisation-
  friendly and have permissive licenses.
- It exposes both a chat-session API and a streaming-generate API, so
  it can be wired into agent loops as a fallback local backend.

## 4. What it is not

Not an agent, not a coding-edit CLI. It is a **local model runtime**
that other CLIs in this catalog (e.g. ones speaking the OpenAI-compat
API) can be pointed at via a small adapter. Compare against
`ollama`, `llamafile`, `llama.cpp`, `lm-studio`, `vllm`,
`sglang` — all of which are in this catalog and solve the same
"run a local model" problem with different tradeoffs.
