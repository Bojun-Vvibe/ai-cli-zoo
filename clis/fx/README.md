# fx

- **Repo:** https://github.com/antonmedv/fx
- **Version:** 39.2.0 (latest stable, 2025-11)
- **License:** MIT ([LICENSE](https://github.com/antonmedv/fx/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install fx` · `go install github.com/antonmedv/fx@latest` · `npm i -g fx` (legacy Node build, kept for back-compat) · `nix run nixpkgs#fx` · static binary releases on the GitHub release page

## What it does

`fx` is an interactive terminal JSON / YAML viewer. You point it at a
file or pipe a stream into it and it opens a full-screen TUI with a
collapsible tree on the left and the path of the current cursor printed
at the bottom of the screen. The product is the keyboard navigation:
arrow keys (or `j` / `k` / `h` / `l` in vim mode) walk siblings and
parents, `e` collapses or expands a subtree, `E` collapses everything,
`.` jumps to the JSON path of the cursor (you can edit the path inline),
`/` opens an incremental fuzzy search across keys and values, `n` / `N`
cycles search hits, and `y` copies the current node — or the path to it,
or the value, depending on the followup key — to the system clipboard.
The bottom-line path expression is the killer feature for shell users:
once you have visually located the field you want, hitting `.` shows
you the exact `.users[3].address.zip` you would type into `jq` or `yq`
to extract it programmatically, eliminating the "guess the path, run
`jq`, see nothing, re-guess" loop that happens with deeply nested
responses. `fx` also accepts a one-shot `--query` (the original Node
version called this a "reduce" function) so you can chain it in a
pipeline (`curl … | fx '.items[] | .id'`), but the interactive TUI is
the reason it exists. Big files are handled via streamed lazy parsing —
opening a 500 MB JSON dump expands one level instantly and only parses
deeper subtrees as you navigate into them. YAML, NDJSON, and JSON5
inputs are auto-detected.

## When to pick it / when not to

Pick `fx` when you already have JSON in front of you and you do not yet
know its shape — a fresh API response, a Lambda CloudWatch log entry, a
Terraform state file, a Kubernetes `get -o json` dump, a `package-lock.json`
you are debugging. The two-pane "tree + path" UX answers "what fields
exist and what do I write to extract this one" in under thirty seconds,
which is exactly the moment a `jq` filter is hardest to author from scratch.
Pair with [`jq`](../jq/) or [`jaq`](../jaq/) for the actual scripted
extraction once you have copied the path out of `fx`, with
[`gron`](../gron/) when you want grep-friendly flat output instead of a
TUI (gron and fx solve the same problem with opposite ergonomics —
gron for `grep | sort | uniq`, fx for arrow keys and clipboard), and
with [`yq`](../yq/) when the source is YAML and you want comment-
preserving edits rather than read-only navigation.

Skip it for non-interactive CI / batch contexts — `jq` / `jaq` / `yq`
are the right tools when no human is present. Skip it for transformation
or aggregation; the one-shot `--query` mode is intentionally limited and
its expression dialect is not `jq`'s, so a complex filter is better
written in `jq` from the start. Skip it on a constrained terminal that
will not render colors or alt-screen properly (CI logs, basic `cat`-style
viewers); the TUI assumes a real terminal. Note that `fx` was rewritten
from Node.js to Go around v30; the `npm` package still installs the
legacy Node build, while `brew` and the GitHub release binaries ship
the Go build — the Go build is faster and has better large-file
handling, and is the one this entry describes. There is also a
**proprietary `fx` Pro** version with extra features behind a license
key; the free OSS build covered here is fully sufficient for the
"explore and extract a path" workflow.

## Example invocations

```bash
# Open a JSON file interactively
fx package.json

# Pipe a stream into fx
curl -s https://api.github.com/repos/cli/cli | fx

# AWS CloudWatch / Lambda response, large nested JSON
aws logs filter-log-events --log-group-name /aws/lambda/my-fn --output json | fx

# Kubernetes resource exploration without remembering the schema
kubectl get pod my-pod -o json | fx

# YAML works the same way
fx values.yaml

# One-shot non-interactive extraction (when you already know the path)
cat orders.json | fx '.orders[] | .id'

# Open NDJSON — fx wraps each line as one document and lets you arrow-key between them
fx access.ndjson

# Inside the TUI keystrokes:
#   .     edit the JSON path expression at the bottom of the screen
#   /     incremental fuzzy search across keys + values
#   e     collapse / expand the cursor node
#   E     collapse everything; e to expand a level
#   y     copy menu — value, path, or whole subtree to the clipboard
#   q     quit
```
