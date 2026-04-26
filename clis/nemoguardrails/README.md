# nemoguardrails

- **Repo:** https://github.com/NVIDIA-NeMo/Guardrails
- **Version:** v0.21.0 (released 2026-03-12)
- **License:** Apache-2.0 (`LICENSE.md` @ SHA `966ad04e1438bb3a6c53a7f7ac057de476d7d297`)

## What it does

NeMo Guardrails is an open-source toolkit for adding programmable safety and
topical rails to LLM-based conversational systems. You author rails in
**Colang** — a small DSL for describing dialogue flows, allowed/disallowed
topics, fact-checking checks, jailbreak detection, and tool-call gating —
and the runtime intercepts every user turn and every model response,
running the rail program before the message reaches the user or the
underlying LLM.

## Install / Run

```sh
pip install nemoguardrails
nemoguardrails chat --config=./config
```

```python
from nemoguardrails import LLMRails, RailsConfig
config = RailsConfig.from_path("./config")
rails = LLMRails(config)
response = rails.generate(messages=[{"role": "user", "content": "Hi"}])
```

A `config/` directory typically contains `config.yml` (model + rails wiring),
one or more `*.co` Colang flow files, and optional knowledge-base docs for
the built-in fact-checking rail.

## AI-native angle

Guardrails-as-code with a real grammar instead of regex blocklists: Colang
flows compose like ordinary control flow, the runtime ships canned rails
for the usual suspects (self-check input/output, hallucination, topical
moderation, tool gating, sensitive-data detection), and every rail is itself
an LLM call you can swap, fine-tune, or replace with a classifier. Plays
well as the policy layer in front of any chat-style agent — including the
agent CLIs catalogued elsewhere in this zoo.
