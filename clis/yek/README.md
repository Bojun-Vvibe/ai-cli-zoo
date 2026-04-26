# yek

- **Repo:** https://github.com/mohsen1/yek
- **Version:** v0.25.0 (latest release)
- **License:** MIT (`LICENSE`)

## What it does

A fast Rust CLI that walks a repository or directory and serializes all
text files into a single stream optimized for LLM context windows. Honors
`.gitignore`, can chunk by token budget, and orders files by git commit
recency so the most-recently-touched code lands closest to the prompt.

## Install

```sh
# Homebrew
brew install yek

# Or from source
cargo install --git https://github.com/mohsen1/yek
```

## Usage

```sh
# Dump current repo as one file, capped at 128k tokens
yek --max-size 128k > context.txt

# Pipe directly into an LLM CLI
yek src/ | llm -m claude-3-5-sonnet "find the auth bug"

# JSON output for programmatic use
yek --json src/ | jq '.files | length'
```

## Why it's interesting

Solves the "stuff a repo into a prompt" problem with three nice
properties: it's fast (Rust + parallel walk), it's gitignore-aware out of
the box, and it sorts by recency so truncation keeps the relevant edits
in window. A common building block paired with `llm`, `aichat`, or any
CLI that takes stdin.
