# llm-jq

- **Repo:** https://github.com/simonw/llm-jq
- **Version:** 0.1.1 (latest release)
- **License:** Apache-2.0 (`LICENSE`)

## What it does

Plugin for Simon Willison's `llm` CLI that turns natural language into a `jq`
program, runs it against piped JSON, and prints both the program and the
result. Lets you say "give me the average price per category" instead of
hand-writing `jq` filters.

## Install

```sh
llm install llm-jq
```

## Usage

```sh
curl -s 'https://api.example.com/data' | llm jq 'top 3 items by score'
```

## Why it's interesting

A small but sharp example of LLM-as-glue: instead of building a chat UI it
embeds the model into a unix pipeline. Pairs especially well with `llm`'s
local model support so you can do JSON wrangling fully offline.
