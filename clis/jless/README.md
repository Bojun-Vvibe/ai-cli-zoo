# jless

> **A command-line JSON and YAML viewer with vim-style navigation**
> — a single Rust binary that opens a JSON / YAML file (or stdin)
> as a collapsible, paginated tree, with `/` regex search, jq-style
> path display, and copy-to-clipboard for any sub-tree. Pinned to
> **v0.9.0** (commit
> `21dd6100edb47145bb9570931bec29697bcd0d9b`,
> [LICENSE](https://github.com/PaulJuliusMartinez/jless/blob/main/LICENSE),
> MIT).

Source: <https://github.com/PaulJuliusMartinez/jless>

## TL;DR

`jless` is what you reach for when `cat foo.json | jq .` produces
40 000 lines of pretty-printed output and you have to find the
one nested field that matters. It opens the file (or stdin) as a
foldable tree, starts with everything collapsed, and lets you
navigate with vim keys (`hjkl`), expand / collapse with
`space` / `enter`, search with `/`, jump to siblings with `J`/`K`,
and read out the current jq path with `y p`. It is a *viewer*,
not a transformer — pair it with [`jq`](https://stedolan.github.io/jq/)
or [`llm-jq`](../llm-jq/) for queries.

## Install

```bash
# Homebrew (macOS / Linux)
brew install jless

# Cargo
cargo install --locked jless

# Linux package managers
# Arch:   yay -S jless     (AUR)
# Nix:    nix-env -iA nixpkgs.jless

# from a release binary (any OS)
curl -Lo jless.zip "https://github.com/PaulJuliusMartinez/jless/releases/download/v0.9.0/jless-v0.9.0-x86_64-apple-darwin.zip"
unzip jless.zip && sudo install jless /usr/local/bin/

# verify
jless --version    # jless 0.9.0
```

No shell hook, no daemon, no config file required. A
`~/.config/jless/config.toml` lets you remap keys and pick a
colour theme.

## License

MIT — see
[LICENSE](https://github.com/PaulJuliusMartinez/jless/blob/main/LICENSE).
Permissive, no attribution required for binaries; redistribute
freely.

## One Concrete Example

```bash
# 1. open a single JSON file
jless big-api-response.json

# 2. pipe from anywhere
curl -s https://api.github.com/repos/rust-lang/rust | jless

# 3. YAML is auto-detected by extension; --yaml forces it from stdin
helm template ./chart | jless --yaml

# 4. start with everything expanded ("data mode" instead of "line mode")
jless --mode data foo.json

# inside jless:
#   j / k             move down / up one row
#   h / l             collapse / expand current node
#   space / enter     toggle expand
#   J / K             jump to next / previous sibling
#   c / e             collapse / expand all children
#   /pattern          regex search forward
#   n / N             next / previous match
#   y p               yank current jq path to clipboard (e.g. ".items[3].metadata.name")
#   y v               yank current value as JSON
#   y k               yank current key
#   .                 hide / show line numbers
#   ? / q             help / quit

# 5. land directly on a path with --path
jless --path '.items[0].spec.containers' deployment.json

# 6. use it as a sanity-check viewer in a pipe with jq
jq '.items[] | select(.status.phase == "Failed")' pods.json | jless

# 7. read a 1 GB JSON dump — jless streams; it does not load it all
jless huge-event-log.json    # cold-open ~200 ms, memory-mapped
```

## Niche It Fills

**JSON / YAML as a foldable tree, not a wall of text.** `jq`
queries when you already know the shape; `less` paginates raw
text but knows nothing about structure. `jless` is the
viewer-layer in between: open something you've never seen
before, fold it down to the schema, drill into one branch,
yank its jq path, paste that into your `jq` query for the
actual transform. Three minutes from "what does this thing
look like" to "I have the path I need".

## Why use it

Three things `jless` does that `cat | jq | less` does not:

1. **Folding by structure, not by line count.** Collapsing an
   object hides all of its children regardless of how many
   thousands of lines they would have rendered to. You can
   browse a 50 000-line JSON file as a 12-row top-level object
   and only expand the branches you care about.
2. **Path yank is the killer feature.** `y p` puts
   `.items[42].spec.template.spec.containers[0].env[3].value`
   on your clipboard; you paste it as the next argument to `jq`,
   `gron`, `python -c "import json; …"`, or your editor's "go to
   key" — no manual counting of array indices, no off-by-one.
3. **Streaming open + memory-map = no parse-everything-up-front
   tax.** A 1 GB JSON dump opens in milliseconds because `jless`
   parses lazily as you scroll; `jq .` on the same file would
   buffer the whole structure first and possibly OOM.

For LLM / agent workflows where the model produces structured
JSON output (tool-call arguments, ndjson event traces, OpenAI
Batch API outputs), `jless trace.json` lets the human reviewer
spot-check shape and content without piping through `jq` for
every angle.

## Vs Already Cataloged

- **Vs [`bat`](../bat/):** orthogonal — `bat` is a syntax-
  highlighting `cat` for any text file (source code, configs,
  logs); `jless` is a structure-aware viewer specifically for
  JSON / YAML. `bat foo.json` shows you 40k pretty-printed
  lines coloured nicely; `jless foo.json` shows you the schema
  folded down. Use `bat` when you want to read top-to-bottom,
  `jless` when you want to navigate a tree.
- **Vs [`llm-jq`](../llm-jq/):** complementary — `llm-jq`
  generates a `jq` query from a natural-language description;
  `jless` is the viewer you open *first* to understand the
  shape, *then* hand the path to `llm-jq` (or write the query
  yourself). The pair: `jless` for "what is this", `llm-jq` for
  "how do I extract X from it".
- **Vs `jq` directly:** `jq` is a transformer / query language
  (DSL with `.`, `|`, `select`, `map`, …); `jless` is a viewer
  with no query language at all. They share a path syntax
  (`.foo[0].bar`) on purpose so the path you yank from `jless`
  drops straight into a `jq` invocation.
- **Vs `fx`, `jnv`, `gron` (not cataloged):** the close peers.
  `fx` is a JS-runtime JSON explorer with mouse + JS expression
  filtering; `jnv` is interactive `jq` with live preview;
  `gron` flattens JSON to grep-able assignment lines. `jless`
  picks the "vim navigation + clipboard-friendly paths + zero
  query language" point in that design space.

## Caveats

- **Viewer only — no edits, no transforms.** You cannot change
  a value, save a modified file, or filter to a sub-tree from
  inside `jless`. For transforms use `jq` (or `jnv` for live
  preview); for edits open the file in your editor.
- **Two modes (line / data) with different keymaps.** Default
  "line mode" is closer to `less`; `--mode data` is the
  fold-everything-by-default tree view. New users sometimes get
  confused that `j` / `k` move differently in the two modes.
  Pick one in `~/.config/jless/config.toml` and stick with it.
- **YAML support is convert-to-JSON-internally.** Comments are
  dropped, key ordering is normalised, and YAML-specific
  features (anchors, multi-doc streams) are flattened. For
  faithful YAML editing use `yq` or your editor; `jless --yaml`
  is for reading.
- **No streaming JSONL / ndjson native mode.** A file that's
  one JSON object per line is treated as malformed; pipe it
  through `jq -s .` (slurp into an array) first, or use
  `jless` on each line via `xargs`.
- **Maintenance has slowed.** Last release v0.9.0 is from
  mid-2023; the project is feature-complete-ish but not
  actively adding things. Open issues stay open; for cutting-
  edge JSON-TUI features, look at `jnv` or `fx`. As a stable
  daily-driver viewer, v0.9.0 is fine and the on-disk binary
  hasn't bitrotted.
