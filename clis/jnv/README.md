# jnv

> **An interactive JSON filter built on `jq`** — opens a
> two-pane TUI where the top pane is a live `jq`-expression
> editor with completion over the actual keys present in the
> loaded document and the bottom pane is the rendered output
> of running that expression against the JSON, re-evaluated on
> every keystroke; supports streaming line-delimited JSON
> (`jq -c` style), saving / loading expression history, and
> exporting the final filter for use in non-interactive
> pipelines. Pinned to **0.7.1** (commit
> `871a8286b984d7c251603fc7d5e1b88eed8ed43b`,
> [LICENSE](https://github.com/ynqa/jnv/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ynqa/jnv>

## TL;DR

`jnv` is the missing REPL for `jq`. The fundamental usability
problem with `jq` has always been that you write the filter
blind — type `.foo.bar.baz`, run it, get `null`, fix a typo,
run it again — and on a 50 KB API response with nested arrays
you spend more time iterating the expression than thinking
about the data. `jnv` flips that: load the file (`jnv
big.json` or `curl … | jnv`), and you get a split-pane TUI
where the top half is your `jq` expression and the bottom half
is the live, prettified output of `jq <that expression>
big.json`, refreshed on each keystroke with debouncing. The
expression editor knows the *actual schema* of the loaded
document and offers tab-completion over keys that exist
("did you mean `.users[].login`, not `.users[].name`?"),
which removes the entire class of `jq: error: Cannot index
object with number` round-trips. When the expression is
right, `Ctrl-c` copies the filter to your clipboard so you
can paste it into the production shell pipeline as
`jq '<that filter>'` without `jnv` in the loop. Streaming
JSONL is supported (`kubectl logs … -o json | jnv`), the
history pane (`Ctrl-r`) recalls past filters, and the whole
thing is one Rust binary with no `jq` runtime dependency
(it embeds [`jaq`](https://github.com/01mf02/jaq) as the
evaluator under the hood, which is faster than `jq` on most
documents and has the same surface syntax).

## When to choose

- **You write `jq` filters by trial and error against large
  responses** — Kubernetes API output, GitHub REST responses,
  Terraform state, AWS CLI `--output json` blobs. The
  per-keystroke re-eval collapses the
  edit/run/edit/run/edit/run loop into a single continuous
  exploration. Once the filter is right, you copy it out and
  use stock `jq` in the script.
- **You don't know the schema of the document** — third-party
  APIs, internal logging blobs, anything with deeply nested
  optional arrays. `jnv`'s key completion shows you what
  paths are reachable from the current cursor position,
  which is faster than reading the docs (or the OpenAPI
  spec, if one exists at all).
- **You're handing JSON to a non-`jq` user** — pair-program
  the filter live, see the output update as you both edit,
  hand them the saved expression. Much less hostile than
  `man jq` for someone who has never written
  `[.[] | select(.status == "open")]` before.
- **You want streaming JSONL exploration** — `tail -f
  app.log.jsonl | jnv` keeps the bottom pane refreshing as
  new lines arrive and the top-pane filter applies to the
  rolling window. Rough equivalent of `jq --unbuffered` plus
  a live preview.
- **You want a single static binary, not a `jq + readline +
  shell-script` rig** — `cargo install jnv`, `brew install
  jnv`, or download the GitHub release tarball. No Python,
  no Node, no `jq` runtime required (the embedded `jaq`
  evaluator handles ~all of `jq`'s grammar).

## Comparable CLIs already in zoo

- **Vs [`jless`](../jless/):** `jless` is a *pager* for JSON
  — `less` for `.json` files, with collapsible nodes,
  `/`-search, and node-path display in the status bar. It
  does not evaluate filters; you navigate the document by
  hand. `jnv` evaluates `jq` expressions live; you navigate
  the document by writing the filter. The natural pipeline
  is `jless` first to *see* the shape, then `jnv` to *carve
  out* the slice you want.
- **Vs [`gron`](../gron/) / [`fx`](../fx/)** (if those land):
  `gron` flattens JSON to grep-able paths
  (`.users[0].login = "alice"`); `fx` is a JS-expression
  interactive viewer. `jnv` is the `jq`-native equivalent
  of `fx` — same TUI shape, but the expression language is
  `jq` (which most ops scripts already use) instead of
  JavaScript.
- **Vs raw [`jq`](https://stedolan.github.io/jq/) /
  `jaq`** (not in zoo as standalone entries): those are the
  evaluators. `jnv` *uses* `jaq` internally and gives you
  the editor on top of it. When the filter is finalized,
  drop `jnv` from the pipeline and call `jq` directly.
- **Vs [`yq`](https://github.com/mikefarah/yq) /
  [`dasel`](https://github.com/TomWright/dasel)** (not in
  zoo): those are multi-format (`yaml` / `toml` / `xml` /
  `csv` / `json`) selectors with no interactive mode.
  `jnv` is JSON-only but interactive; pick the right tool
  for the format you actually have.

## Caveats

- **JSON-only.** YAML, TOML, XML are not supported — convert
  first (`yq -o=json . file.yaml | jnv`). The `jq` grammar
  is JSON-shaped and `jnv` does not pretend otherwise.
- **The embedded evaluator is `jaq`, not `jq` proper.**
  `jaq` covers the vast majority of the `jq` language but
  has documented gaps (some date functions, a few regex
  edge cases, `input_filename`). If your filter relies on
  one of those, prototype in `jnv` and confirm in the real
  `jq` before shipping. The coverage is good enough that
  you will rarely notice.
- **Very large documents are slow per keystroke.** The TUI
  re-evaluates on every keypress; on a 200 MB JSON file the
  per-keystroke cost is visible. Pre-filter with `jq -c
  '.relevant_field' big.json | jnv` to shrink the working
  set before opening the interactive editor.
- **Streaming JSONL has a default buffer cap.** `jnv` does
  not retain every line of a `kubectl logs -f` stream
  forever — old lines age out. For "filter the entire log
  history" use the static-file mode after collecting the
  log; for "filter the live tail" the streaming mode is
  what you want.
- **Clipboard integration depends on the OS clipboard
  daemon.** macOS works out of the box (`pbcopy`); Linux
  needs `xclip` / `wl-clipboard` installed; over SSH
  without an X forward / OSC52-aware terminal the copy
  silently no-ops — fall back to `Ctrl-s` to write the
  filter to a file.
