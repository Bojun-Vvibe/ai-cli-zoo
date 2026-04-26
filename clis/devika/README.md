# devika

- **Repo:** https://github.com/stitionai/devika
- **Version:** unreleased (no tags); pinned to commit `80bb343` (default branch)
- **License:** MIT (`LICENSE`)
- **Language:** Python (backend) + Svelte (UI)
- **Install:** `git clone https://github.com/stitionai/devika && cd devika && pip install -r requirements.txt && python devika.py`

## One-line summary

An agentic AI software engineer that takes a high-level objective, plans it,
researches the web, and writes code — open-source answer to Devin.

## What it does

Devika positions itself as a self-hosted alternative to closed agentic-coder
products. The architecture splits responsibilities across discrete modules:

- **Agent planner** decomposes the objective into steps
- **Researcher** drives a Playwright browser to search and read pages
- **Coder / patcher** writes and edits files in a project workspace
- **Knowledge base** stores retrieved snippets per project for grounding
- **Browser interaction** is observable in the bundled Svelte UI

It supports Claude, GPT-4, Gemini, Mistral, Groq, and local models via
Ollama. State per project (plans, code, browser history) is persisted to
SQLite so a session is resumable.

```bash
# After install, browse to http://localhost:3001
# Create a project, pick a model, type an objective, e.g.:
#   "Build a Flask app that surfaces Hacker News top stories with sentiment."
```

## When to choose it

- You want a **visible, inspectable** agent loop (browser pane + plan pane)
  instead of an opaque CLI stream.
- You want a **self-hostable Devin-shaped** product and are OK running a
  Python backend + Svelte frontend stack.
- You want MIT-licensed code you can fork and rewire.

## When NOT to choose it

- You need active upstream maintenance — the project's release cadence has
  slowed; treat it as a reference architecture, not a daily driver.
- You want a terminal-only workflow — Devika's value is in the UI; without
  it you lose half the product.
- You need production SLAs — there are no tagged releases; you're pinning
  to commits.
