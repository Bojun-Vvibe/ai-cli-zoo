# files-to-prompt

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/files-to-prompt>
> Binary name: `files-to-prompt`

`files-to-prompt` is the **context-packaging primitive** of the
catalog. It is not an LLM client at all — it does not talk to any
provider, it has no model selector, no streaming, no chat. Its only
job is to take a list of paths (files, directories, glob patterns,
or `-` for stdin) and emit a single text blob that you then pipe
into something that *is* an LLM client. The dominant pairing in the
wild is `files-to-prompt ./src | llm -m claude-sonnet 'review this'`
or the same pipe into `mods`, `sgpt`, `aichat`, `fabric`, or any
other stdin-consuming entry in this catalog.

```
files-to-prompt src/ docs/ -e py -e md --cxml | llm -m gpt-5 'find bugs'
files-to-prompt . --ignore '*.lock' --ignore-files-only -c | pbcopy
files-to-prompt my_module/ -o context.txt
```

This is taxonomically distinct from `aider` (which builds its own
repo-map internally and never exposes it as text), from `aichat`
(which has a built-in RAG store keyed on embeddings), and from
agent CLIs like `opencode` / `claude-code` / `codex` (which do
ad-hoc file reads inside their tool loops). `files-to-prompt`'s
contract is **deterministic, line-by-line reproducible context
construction** that you can `diff`, version, and pipe.

## 1. Install footprint

- Pure Python, single dependency (`click`). Ships on PyPI as
  `files-to-prompt`. Install with `pipx install files-to-prompt`,
  `uv tool install files-to-prompt`, `pip install --user
  files-to-prompt`, or `brew install files-to-prompt`.
- One executable on `$PATH`. No config file, no state directory, no
  network calls — the binary never opens a socket.
- Honors `.gitignore` by default (toggleable with
  `--ignore-gitignore`); also reads `.gitignore`-style files
  recursively from each directory it descends into.

## 2. License

Apache-2.0.

## 3. Models supported

**None.** This tool does not talk to any model. Output is plain text
(or XML, or markdown) intended for downstream consumption by another
CLI or by paste into a chat UI. The pairing tool decides the model.

The "supported models" question collapses to "supported *output
formats* a downstream model will parse cleanly":

- **Default** — repeated `path/to/file.py:\n---\n<contents>\n---\n`
  blocks. Works fine for any model.
- **`--cxml`** — Anthropic-recommended XML wrapping
  (`<documents><document index="1"><source>...</source>
  <document_content>...</document_content></document></documents>`).
  Claude models are explicitly tuned to attend to this shape.
- **`--markdown`** / **`-m`** — fenced code blocks with the language
  inferred from extension. Works well with GPT-class models and is
  the friendliest shape for human re-reading.
- **`--line-numbers`** / **`-n`** — prefix every line with its
  number, so the model can cite `path:42` accurately. Pairs naturally
  with code-review prompts.

## 4. MCP support

None. No client, no server, no tool-call surface. `files-to-prompt`
is one layer below MCP — it produces the *content* that a downstream
agent might otherwise fetch via an MCP filesystem server. If you
already have an agent CLI with MCP filesystem access, you usually do
not also need `files-to-prompt`. The two-tool pipeline is for
**non-agent** workflows where you want a single round-trip to a chat
endpoint.

## 5. Sub-agent model

None. There is no agent loop, no tool call, no second pass. Each
invocation walks the filesystem once, writes one blob, exits.

The composition primitive is the **shell pipe**, identical in shape
to the `mods` / `llm` / `fabric` / `sgpt` family:

```
files-to-prompt src/ -e py --cxml \
  | llm -m claude-opus 'summarize the public API surface' \
  | files-to-prompt - docs/api.md --cxml \
  | llm -m claude-opus 'reconcile the summary with the existing docs'
```

Two model calls, two context-packing steps, no framework.

## 6. Telemetry stance

**Off, with no opt-in.** The binary makes no network calls. It reads
files, writes stdout, exits. The downstream LLM client obviously
sees the packed context.

## 7. Prompt-cache strategy

None at this layer (no model is called). However, because output is
**deterministic** for a given input set + flags, downstream
provider-side prefix caching (Anthropic explicit `cache_control`,
OpenAI implicit, Gemini implicit) works extremely well: the same
`files-to-prompt src/ --cxml` output can be reused as a cached
prefix across many follow-up questions in the same hour, dropping
re-ingest cost dramatically.

The standard Anthropic-cache pattern is

```
files-to-prompt src/ --cxml > /tmp/ctx.xml
llm -m claude-sonnet --cache 'first question' < /tmp/ctx.xml
llm -m claude-sonnet --cache 'second question' < /tmp/ctx.xml
```

— two calls share one cached prefix.

## 8. Hot keybinds

There is no TUI. Relevant flags:

- `files-to-prompt <paths...>` — accepts any number of files,
  directories, or glob patterns. `-` reads paths from stdin
  (one per line), which composes with `find`, `fd`, `git ls-files`,
  `rg --files`.
- `-e <ext>` / `--extension <ext>` — restrict to one or more
  extensions. Repeatable: `-e py -e md`.
- `--ignore <pattern>` — gitignore-style exclusion. Repeatable.
- `--ignore-gitignore` — disable the default `.gitignore` honoring.
- `--ignore-files-only` — apply `--ignore` to files only, not to
  directory pruning.
- `-c` / `--cxml` — Anthropic XML output.
- `-m` / `--markdown` — fenced markdown output.
- `-n` / `--line-numbers` — prefix every content line with its 1-based
  line number.
- `-o <file>` / `--output <file>` — write to file instead of stdout.
  Useful when the result is multi-MB and you want to avoid terminal
  scroll-jank.
- `--null` / `-0` — when reading paths from stdin, expect NUL
  separators. Pairs with `find -print0` and `git ls-files -z`.

The single most useful idiom is `git ls-files -z | files-to-prompt
- --null --cxml | pbcopy` — pack every tracked file in the current
repo and stash on the clipboard, ready to paste into any chat UI.

## 9. Killer feature, weakness, when to choose

**Killer feature.** A **deterministic, scriptable, format-agnostic
context-packing primitive** that respects `.gitignore` by default
and emits exactly the wrapper shape Anthropic / OpenAI / Gemini
documentation recommends. Output is reproducible byte-for-byte for a
given input set, which means you can `diff` two context blobs to
explain why an LLM gave a different answer between two runs. Nothing
else in the catalog occupies this layer cleanly — competitors are
either too smart (an agent that decides which files to read) or too
dumb (`cat`, which loses paths and ignores `.gitignore`).

**Weakness.** Three:

1. **No model loop.** This is a strength architecturally and a
   weakness ergonomically — beginners who type `files-to-prompt
   src/` and expect an answer get a 200 KB blob on stdout instead.
   You always need a second tool. The README is explicit about this
   but the surprise is real.
2. **No token counting or budgeting.** It will happily emit a 5 MB
   blob that exceeds your model's context window. Pair with
   [`ttok`](https://github.com/simonw/ttok) (same author) or with
   `--output` plus `wc -c` and a sanity check.
3. **No semantic ranking.** Every matching file is included; there
   is no embedding-based selection, no relevance scoring, no
   smart-truncation. If your repo is too big to fit, you must
   pre-filter with `-e`, `--ignore`, or upstream `git ls-files |
   grep`. For embedding-driven selection use [`aichat`](../aichat/)'s
   built-in RAG instead.

**When to choose.**

- You want to **paste a whole subtree** into a chat UI in one shot,
  with file paths preserved and `.gitignore` honored.
- You are scripting a **one-shot LLM call** over a known file set
  (CI, cron, a `Makefile` target) and want reproducible context.
- You are using a **non-agent** LLM CLI (`llm`, `mods`, `sgpt`,
  `fabric`, `aichat`) and need to feed it more than one file at
  once.
- You want to take **deliberate advantage of provider-side prefix
  caching** by ensuring the same prefix bytes go up on every call.
- You need an **auditable diff** of "what context did the model
  actually see" — you can save the blob to disk and check it in.

**When not to choose.**

- You want the tool to **decide which files matter**. Use an agent
  CLI with file-reading tools: `aider` (repo-map), `opencode`,
  `codex`, `claude-code`, or `cline`.
- You need **embedding-based retrieval over a folder**. Use
  [`aichat`](../aichat/) (built-in RAG over a session) or build on
  top of [`llm`](../llm/) plus `llm-embed` plugins.
- You want a **chat session that remembers** the packed context
  across turns. Pipe `files-to-prompt` output once into [`llm
  chat`](../llm/) and let `llm` hold the conversation, or use
  [`aichat`](../aichat/)'s `.file` REPL command.
- You want **agentic file edits** based on the packed context.
  `files-to-prompt` is read-only in spirit — it can pack a tree but
  cannot apply diffs back. Use [`aider`](../aider/) for that loop.
