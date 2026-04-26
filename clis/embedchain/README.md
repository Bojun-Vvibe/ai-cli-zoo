# embedchain

- **Repo:** https://github.com/embedchain/embedchain
- **Version:** ts-v3.0.2 (latest release)
- **License:** Apache-2.0 (`LICENSE` @ SHA `d20d5102c3cf97ecbee54afd65893de4a11d26fe`)

## What it does

A "memory layer for AI agents" — a high-level framework that lets you ingest
documents, URLs, YouTube transcripts, PDFs, and more into a vector store and
then query them through a single `App` object. Handles chunking, embedding,
and retrieval so you can build a RAG-backed assistant in a handful of lines.

## Install / Run

```sh
pip install embedchain
```

```python
from embedchain import App
app = App()
app.add("https://en.wikipedia.org/wiki/Elon_Musk")
print(app.query("How many companies does Elon Musk run?"))
```

A TypeScript port lives under the same repo (`ts-v*` release tags).

## AI-native angle

Embedchain leans into the "agent memory" framing: pluggable vector stores,
LLM providers, and embedding models behind one tiny API. Useful as a
batteries-included starting point when you want RAG without wiring up
loaders, splitters, and retrievers by hand.
