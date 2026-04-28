# jqp

- **Repo:** https://github.com/noahgorstein/jqp
- **Version:** v0.8.0 (2025-09-28)
- **License:** MIT ([LICENSE](https://github.com/noahgorstein/jqp/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install jqp` · `go install github.com/noahgorstein/jqp@latest` · prebuilt binaries on the GitHub release page · binary name is `jqp`

## What it does

`jqp` is a TUI playground for `jq`. You feed it a JSON document
(file argument, stdin pipe, or paste-into-buffer) and it opens a
three-pane terminal interface: the **input JSON** on the left,
your **`jq` query** at the top, and the **live-evaluated output**
on the right that updates on every keystroke. Syntax highlighting
on both JSON sides, line numbers, fold/unfold of nested objects,
search inside the input, and an in-app cheatsheet (`?` key) covering
the operators you forget every six months (`|`, `//`, `as`, `reduce`,
`foreach`, slice indices, `to_entries`).

When you have the query you want, `Ctrl+S` saves it to a file or
copies the formatted output to the clipboard so you can paste it
straight into a script. Themes follow your terminal background
(light / dark variants of Dracula, GitHub, etc.) via a config
file at `~/.config/jqp/config.yaml`.

## When to pick it / when not to

Reach for `jqp` when you have a **non-trivial JSON document and an
exploratory query**: an unfamiliar API response, a deeply nested
log dump, a Terraform state file, a `kubectl get -o json` blob.
The keystroke-by-keystroke feedback loop turns "iterate `jq -r '…'`
in the shell, run, scroll, edit history, run again" into one
sub-second loop, which matters most when you do not yet know the
shape of the data.

Skip it for one-shot pipelines you already know (`jq -r '.items[].name'`
in a shell is shorter than launching a TUI). Skip when the JSON is
huge enough that `jq` itself is the bottleneck — `jqp` re-evaluates
on every keystroke. Skip on a remote machine over a slow SSH link;
the redraw cost adds up.

## Why it matters in an AI-native workflow

LLM tool calls and agent traces produce JSON constantly:
OpenAI / Anthropic message logs, MCP tool responses, OpenTelemetry
spans serialised as JSON Lines, evaluation result blobs. The shape
is rarely documented and rarely stable across model versions, so
the first thing a human does when triaging "why did the agent do
that?" is squint at a JSON tree until the right `.choices[0].message.tool_calls[0].function.arguments`
path becomes clear. `jqp` collapses that triage loop from minutes
to seconds and makes the resulting `jq` filter directly copy-pastable
into the shell pipeline that consumes the trace in CI.

## Example invocations

```bash
# Open a JSON file in the playground
jqp -f response.json

# Pipe in from a network call
curl -s https://api.github.com/repos/noahgorstein/jqp | jqp

# Pipe a kubectl dump and explore it interactively
kubectl get pods -o json | jqp

# Start with a pre-filled query
jqp -q '.items[] | {name, status: .status.phase}' -f pods.json

# Inside the TUI:
#   ? — cheatsheet
#   /  — search the input JSON
#   Ctrl+S — save the query / output
#   Ctrl+C — copy the output to clipboard
#   Tab — cycle focus between query / input / output panes
```

## Alternatives in this catalog

- [`jnv`](../jnv/) — also a `jq` TUI; jnv leans on a tree-navigator
  metaphor (arrow-key drill into the JSON, query is built for you),
  jqp leans on a free-form query editor with live preview. Pick jnv
  when you do not know `jq` syntax yet, jqp when you do.
- [`jq`](../jq/) / [`gojq`](https://github.com/itchyny/gojq) — the
  underlying engines; jqp embeds gojq so behaviour matches.
- [`xq`](../xq/) — sibling story for XML / HTML when the document
  is not JSON.
- [`fx`](../fx/) — another interactive JSON viewer with a different
  UI philosophy (mouse-driven node folding rather than live query).
