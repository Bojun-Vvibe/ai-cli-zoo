# burr

- **Repo:** https://github.com/apache/burr
- **Version:** `burr-0.40.2` (latest release)
- **License:** Apache-2.0 (`LICENSE`)
- **Language:** Python
- **Install:** `pip install "burr[start]"` then `burr` (launches the local UI)

## One-line summary

A library + CLI for building LLM applications as **explicit state machines**,
with a free local telemetry/inspector UI that records every transition for
replay and debugging.

## What it does

Burr models an LLM app as a graph of `Action` nodes that read/write a typed
`State` and choose the next node via `Transition` predicates. The CLI side
is small but load-bearing:

- `burr` — boots the local web UI on `http://localhost:7241` that streams
  every action invocation, state diff, and prompt/response pair from any
  Burr app pointed at the local tracker.
- `burr init` — scaffolds a starter project with a state machine, a
  tracker, and a runnable example.
- `burr-test-case create` — captures a recorded trace and freezes it as a
  pytest fixture so you can regression-test a single action against its
  exact historical inputs.

The pattern is opinionated: every step is a pure-ish function of state,
transitions are declarative, and the tracker is the source of truth for
"what happened on this run" — which makes Burr genuinely useful for
auditing agent loops that keep going off the rails.

```bash
pip install "burr[start]"
burr                          # opens the local inspector UI
burr-test-case create \
  --project-name my-agent \
  --partition-key user-42 \
  --app-id <uuid-from-ui> \
  --sequence-id 7 \
  --target-file-name tests/fixtures/step7.json
```

## When to choose it

- You're tired of agent runs being a black box and want a **deterministic,
  replayable trace** of every state transition without paying for a hosted
  observability vendor.
- Your "agent" is really a **workflow with branching** (router → tool →
  reflect → retry) and you want to see the graph instead of guessing it
  from log lines.
- You want **regression tests on individual steps** of an LLM app, captured
  from real traffic rather than hand-written.

## When NOT to choose it

- You want a one-shot coding agent that edits files — Burr is a framework
  for building those, not one itself. Reach for `aider` / `codex` /
  `opencode` instead.
- You're allergic to defining state schemas up front; Burr's value
  evaporates if every action just mutates a free-form dict.
- You need a hosted, multi-tenant trace backend out of the box — the local
  tracker is the default; the hosted variant is a separate product.
