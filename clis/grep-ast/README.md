# grep-ast

> Snapshot date: 2026-04. Upstream: <https://github.com/Aider-AI/grep-ast>
> Pinned version: **v0.8.1** (read from `setup.py` on `main`; the repo
> publishes to PyPI as `grep-ast` but does not cut GitHub releases).

A tiny `grep`-shaped CLI that searches source files **with AST context
attached**. For every regex hit it prints the matching line plus the
enclosing function / class / block, derived from a tree-sitter parse,
so the LLM (or human) sees a usable slice instead of one stranded line.
Maintained by the aider project (`Aider-AI`); the AST surface inside
aider's repo-map is a near-relative of this code.

## 1. Install footprint

- `pipx install grep-ast` (recommended) or `pip install grep-ast`.
- Pulls `tree-sitter`, `tree-sitter-language-pack` and `pathspec`
  (~25 MB total — most of it is the bundled language grammars).
- Installs two console scripts: `grep-ast` and `gast` (alias).
- Python 3.9+. macOS, Linux, Windows.
- No daemon, no config file, no state.

## 2. License

Apache-2.0 (verified from the upstream `LICENSE` file and the
`License :: OSI Approved :: Apache Software License` classifier in
`setup.py`).

## 3. Models supported

**None.** `grep-ast` is a pure local AST/regex tool — no LLM call, no
network. It emits text designed to be piped into a downstream LLM CLI
(`mods`, `llm`, `aichat`, `aider /run`, `claude-code` via `!` slash).

## 4. MCP support

**No.** Stdin/stdout only. Wrap it in a thin MCP server yourself if you
want an agent to call it as a tool — three lines of `mcp` Python.

## 5. Sub-agent model

None. One regex, one walk over the supplied paths, one batched output.

## 6. Telemetry stance

**Off.** No analytics, no network calls. The only file-system writes
are stdout and stderr.

## 7. Prompt-cache strategy

N/A — this tool never talks to a model.

## 8. Hot keybinds (CLI flags)

`grep-ast` is a one-shot CLI, not a REPL. The flags that matter:

| Flag | Effect |
|------|--------|
| `--encoding <enc>` | Override file decoding (default UTF-8) |
| `--languages` | Print the list of tree-sitter languages it knows |
| `--ignore-case` / `-i` | Case-insensitive regex |
| `--color` / `--no-color` | Force ANSI colour on/off |
| `--line-number` / `-n` | Prefix `path:line:` like `grep -n` |
| `--verbose` / `-v` | Print parser diagnostics |

Example: `gast 'def authenticate' app/` returns each `def authenticate`
line **plus the enclosing class body**, not just the matched line.

## 9. Killer feature, weakness, when to choose

- **Killer:** **AST-anchored slices instead of line-bound `grep`
  output.** `grep -n 'BACKOFF_MS' src/` returns a hundred bare line
  numbers; `gast 'BACKOFF_MS' src/` returns each function or method
  body that mentions `BACKOFF_MS`, with the function signature on top —
  exactly the shape an LLM needs to actually answer "what changes if I
  bump this constant?" without you separately `cat`-ing each file.
- **Weakness:** read-only. No editing, no symbol rename, no semantic
  understanding beyond what tree-sitter produces. For real symbolic
  edits (rename across files, find-references) reach for a real LSP
  surface like [serena](../serena/).
- **Choose it when:** you want a context packer between `rg` and a
  full repo-map — when [files-to-prompt](../files-to-prompt/) is too
  coarse (whole files), [symbex](../symbex/) is too narrow (Python only,
  exact symbols), and you want "every place that mentions X, with the
  enclosing function" in one short pipe.
