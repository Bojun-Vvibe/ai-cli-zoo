# llm-openrouter

- **Repo:** https://github.com/simonw/llm-openrouter
- **Version:** 0.6 (latest release)
- **License:** Apache-2.0 (`LICENSE`)

## What it does

Plugin for the `llm` CLI that adds support for every model hosted on
OpenRouter — hundreds of frontier and open-weights models behind a single
API key. After installing, `llm models` lists each OpenRouter model and
you can use them with `llm -m openrouter/<id> "..."`.

## Install

```sh
llm install llm-openrouter
llm keys set openrouter
```

## Usage

```sh
llm -m openrouter/anthropic/claude-sonnet-4.5 "explain CAP theorem"
llm -m openrouter/meta-llama/llama-3.3-70b-instruct "summarise: $(cat doc.md)"
```

## Why it's interesting

Turns the `llm` CLI into a multi-provider playground without juggling SDKs
or env vars per provider. Useful for cheap A/B comparison across model
families from the same shell prompt or script.
