# datasette

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/datasette>
> Version pinned at write-time: **0.65.2**
> License: **Apache-2.0** (file: [`LICENSE`](https://github.com/simonw/datasette/blob/main/LICENSE))

`datasette` is Simon Willison's open-source multi-tool for **exploring and
publishing data** as a queryable JSON API + browseable UI on top of SQLite.
It is included in this catalog because, although it predates the LLM wave, it
has quietly become the connective tissue under several of Simon's AI CLIs:
the `llm` tool stores its log in a SQLite database that `datasette` is
designed to inspect, and the broader `datasette` plugin ecosystem now
includes LLM-powered query helpers and embeddings browsers.

It is the right tool when you have **already accumulated logs, embeddings, or
scraped data in SQLite** and want to point an LLM (or a human) at the result
without standing up Postgres + a web framework.

## 1. Install

```bash
# pipx (recommended)
pipx install datasette

# or pip in a venv
pip install datasette

# Homebrew
brew install datasette
```

Single Python package, ~10 MB. No daemon. Plugins install via
`datasette install <plugin>`.

## 2. Simple invocation

```bash
# Serve any sqlite file as a browseable JSON API at http://127.0.0.1:8001
datasette my-data.db

# Inspect the llm CLI's log database (the integration point most agent
# operators actually use)
datasette "$(llm logs path)"

# One-off SQL from the shell, JSON out
datasette --get "/my-data/users.json?_shape=array&_size=10" my-data.db
```

## 3. Why it lives in this catalog

- It is the canonical viewer for `llm`'s log database, so anyone running
  `llm` at scale ends up running `datasette` too.
- The plugin surface (`datasette-llm-embed`, `datasette-extract`,
  `datasette-enrichments-gpt4`) turns it into a lightweight RAG
  inspection tool: search embeddings, run an LLM enrichment pass over a
  column, view results.
- It is the smallest "publish a queryable dataset" surface that does not
  require a separate database server.

## 4. What it is not

Not an agent, not a coding CLI, not a chat client. Listed here purely
because of its tight integration with the `llm`-family CLIs already in
this catalog and because plugin authors increasingly use it as the
shipping target for LLM-derived datasets.
