# pieces-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/pieces-app/cli-agent>
> Last verified release: **1.20.1** (2026-02-23, tag SHA `bfa55bee345c`).
> License file: [`LICENSE`](https://github.com/pieces-app/cli-agent/blob/main/LICENSE) — MIT.

A Python CLI front-end to **PiecesOS**, a long-running local daemon
that indexes the developer's recent activity (clipboard snippets,
saved code blocks, IDE selections) into a personal corpus and answers
chat questions grounded in that corpus. The CLI exposes
`pieces save`, `pieces list`, `pieces search`, `pieces ask`, and
`pieces chat` — i.e. capture-and-recall plus a RAG-shaped chat over
your own snippet history.

It earns its catalog slot as the only entry whose primary input
*shape* is "code I have personally touched recently", as opposed to
"a repo on disk" (`aider`, `repomix`), "a curated library of
prompts" (`fabric`), or "a file I just `curl`'d" (`strip-tags`). The
CLI itself is open source (MIT); the underlying PiecesOS daemon it
talks to is a closed-source local binary.

## 1. Install

```bash
pip install pieces-cli
# Then start PiecesOS (the local daemon, separate download from
# pieces.app); the CLI talks to it on http://localhost:39300.
pieces onboarding
```

## 2. Example

```bash
# Save the last clipboard entry with a tag:
pieces save -c

# Ask a question grounded in your saved snippets:
pieces ask "what was that ffmpeg command I used for hardware encoding?"

# Interactive chat with snippet citations:
pieces chat
```

## 3. Honest limitation

The CLI is MIT and open, but the **PiecesOS daemon it requires is a
closed-source binary** distributed by Pieces for Developers Inc. You
cannot run `pieces-cli` against a self-hosted backend, you cannot
audit how the snippet corpus is stored locally, and the chat models
default to Pieces-routed cloud endpoints (configurable to local
Ollama, but the routing logic itself is opaque). Treat this as a
convenience wrapper around a vendor daemon, not a self-contained
open-source tool like `khoj` or `txtai`.
