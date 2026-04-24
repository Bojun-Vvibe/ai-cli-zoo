# repomix

> Snapshot date: 2026-04. Upstream: <https://github.com/yamadashy/repomix>
> Binary name: `repomix`

`repomix` is the **whole-repo packer with remote fetch and token
accounting** of the catalog. It overlaps in spirit with
[`files-to-prompt`](../files-to-prompt/) (both pack a directory
into one blob suitable for pasting into an LLM) but the contract is
different: `files-to-prompt` is a Python Unix filter that emits one
text shape (XML or markdown) and respects `.gitignore`; `repomix`
is a Node CLI that **fetches remote GitHub repos directly**, emits
**multiple structured formats** (XML, markdown, plain text), counts
**per-file and total tokens** with the real `tiktoken` tokenizer,
runs an optional **secret scan** (Secretlint) before writing, and
optionally **compresses code via Tree-sitter** to drop function
bodies while keeping signatures.

That feature combination — remote-fetch + token counts + secret
scan + structured compression — is the differentiator. None of the
other context-packing entries in this catalog do all four.

The four canonical invocations:

- `repomix` — pack the current directory into `repomix-output.xml`.
- `repomix --remote yamadashy/repomix` — clone a public repo to a
  temp dir, pack it, write the output locally, clean up.
- `repomix --compress` — run Tree-sitter compression: keep imports,
  class/function signatures, top-level constants; drop function
  bodies. Typical 5–10× reduction on real codebases.
- `repomix --style markdown --output ctx.md --top-files-len 20` —
  markdown output with a "20 largest files" summary at the top.

There is also `repomix init` (write a `repomix.config.json`),
`--include` / `--ignore` (glob filters layered on top of
`.gitignore`), `--instruction-file-path` (prepend a custom prompt
header), and an MCP-server mode (`repomix --mcp`) that exposes
packing as an MCP tool for `claude-code`, `opencode`, or any
MCP-aware client to call.

## 1. Install footprint

- Node CLI, distributed via npm. ~30 MB installed (Node module
  graph + `tiktoken` + `tree-sitter` parsers + Secretlint rules).
- `npm install -g repomix`, `npx repomix`, or
  `brew install repomix`. A Docker image (`ghcr.io/yamadashy/
  repomix`) is published for sandbox use.
- Configurable via `repomix.config.json` at the repo root, with
  `repomix init` scaffolding the file. Per-user defaults live at
  `~/.config/repomix/repomix.config.json`.
- Requires Node 20+. The Tree-sitter compression path pulls in
  precompiled grammars for ~20 languages (TS/JS, Python, Go, Rust,
  Java, C/C++, Ruby, PHP, etc.); enabling `--compress` adds a
  one-time grammar download on first use.

## 2. License

MIT.

## 3. Models supported

**None directly.** `repomix` is a context packer, not a model
client — same posture as [`files-to-prompt`](../files-to-prompt/)
and [`symbex`](../symbex/). It produces a blob of text designed to
be the *input* to a model call you make with another tool
([`llm`](../llm/), [`mods`](../mods/), [`claude-code`](../claude-code/)
in `-p` mode, web chat paste, etc.).

It does, however, know about *tokenizers*: the `--token-count-tree`
and per-file token counts use `tiktoken` (OpenAI-style BPE) by
default and can be switched to `o200k_base` (GPT-4o family) or
`cl100k_base` (GPT-3.5/4) via config. There is no Anthropic or
Gemini tokenizer; the count is a "good enough" estimate for
budget planning across providers.

## 4. MCP support

**Yes — server side.** `repomix --mcp` starts an MCP server (stdio
transport) that exposes `pack_codebase`, `pack_remote_repository`,
`read_repomix_output`, `grep_repomix_output`, and
`file_system_read_file` as MCP tools. Wire it into
[`claude-code`](../claude-code/), [`opencode`](../opencode/),
[`cline`](../cline/), or any other MCP-aware client and the agent
can pack subtrees on demand instead of you pre-baking a context
file. There is no MCP client side — `repomix` does not consume
other MCP servers.

## 5. Sub-agent model

None. `repomix` is a one-shot CLI: scan files → tokenize → render →
write. There is no agent loop, no LLM call, no recursion. The
"intelligence" is entirely in (a) the Tree-sitter compression pass
and (b) the Secretlint secret-scanning pass; both are static.

## 6. Telemetry stance

**Off, with no opt-in.** `repomix` makes no analytics calls. The
only network egress in normal operation is `git clone` against
`github.com` (or any other host) when you pass `--remote`, and the
one-time Tree-sitter grammar download on first `--compress` use.
Both are visible in `--verbose` output. No daemon, no background
sync, no usage reporting.

The Secretlint pass is a strict positive: by default `repomix`
**refuses to write** any file it believes contains a secret (AWS
keys, GitHub tokens, private SSH keys, etc.) and prints the offending
file/line. Disable with `--no-security-check` if you understand the
risk. This is the single biggest reason to prefer `repomix` over
`files-to-prompt` in a workflow where the packed blob will be
pasted into a third-party chat UI.

## 7. Prompt-cache strategy

Not applicable directly — `repomix` does not call models. The
*output* shape is, however, deliberately **prefix-stable**: the
file ordering is deterministic (alphabetical within each directory
by default, configurable), the header block is identical across
runs, and only the file-content sections differ when files change.
This means downstream Anthropic / Gemini prompt-prefix caching
gets maximum hit rate when you re-run `repomix` on a repo that
changed in only a few files.

## 8. Hot keybinds

There is no TUI, no REPL, no interactive picker. Ergonomics live
in the flag set:

- `repomix --compress` — Tree-sitter compression; usually the first
  flag to add when the unpacked output blows your context budget.
- `repomix --remote <user/repo>` — pack a public GitHub repo
  without cloning it yourself. `--remote-branch <ref>` pins to a
  branch, tag, or commit SHA.
- `repomix --include "src/**/*.ts" --ignore "**/*.test.ts"` —
  glob filters layered on top of `.gitignore` and the built-in
  default-ignore list.
- `repomix --style markdown` / `--style xml` / `--style plain` —
  output shape. XML is the Anthropic-recommended shape; markdown
  reads better as a paste into a chat UI; plain is for downstream
  tools that do their own parsing.
- `repomix --token-count-tree` — print a tree view of per-directory
  token totals before writing the output, so you can iterate on
  `--include` until the count fits your budget.
- `repomix --copy` — also copy the output to the system clipboard
  (uses `clipboardy` under the hood; macOS / Linux / Windows).
- `repomix --output-show-line-numbers` — prefix each source line
  with its line number, matching the shape `aider` and
  `claude-code` use internally for diff anchoring.

## 9. Killer feature, weakness, when to choose

**Killer feature.** `--remote` + `--compress` + Secretlint is the
combination no other entry in the catalog has. You can:

```
repomix --remote some-org/some-lib --compress --token-count-tree
```

…and in one command get a token-budgeted, secret-scanned, signature-
only pack of a repo you do not even have cloned, ready to paste into
a chat UI to ask "what does this library actually do?". No `git
clone`, no `pip install`, no manual file curation, no risk of
leaking the maintainer's accidentally-committed AWS key into your
chat history.

**Weakness.** Three:

1. **Heavyweight install for a "just pack files" job.** ~30 MB of
   Node modules, plus on-demand Tree-sitter grammars. If you only
   ever pack local Python files, [`files-to-prompt`](../files-to-prompt/)
   is a 200 KB pure-Python install that does the same job (without
   the secret-scan and remote-fetch features).
2. **Tokenizer is OpenAI-only.** The token counts are a useful
   estimate for Claude / Gemini but not exact. If you are squeezing
   the last 5% of a context window for a specific provider, count
   tokens with that provider's own tokenizer.
3. **Compression is lossy by design.** `--compress` drops function
   bodies. For a "explain this library's API" prompt that is
   exactly right; for a "find the bug in this function" prompt it
   is exactly wrong. Know which question you are asking before
   adding the flag.

**When to choose.**

- **You want to ask a question about a public repo you do not
  have checked out.** `repomix --remote owner/repo --compress` is
  the path. No clone, no setup.
- **You are pasting context into a third-party chat UI** (web
  Claude, web ChatGPT, web Gemini) and you want the secret-scan
  safety net. The Secretlint pass has caught real keys in real
  repos; it is worth the 30 MB.
- **You want to feed an MCP-aware agent** (`claude-code`,
  `opencode`, `cline`) the ability to pack arbitrary subtrees on
  demand. `repomix --mcp` is the cleanest server you can wire in
  for that.
- **You need per-directory token budgets** to plan what to send.
  `--token-count-tree` is the only tool in the catalog that gives
  you that view directly.

**When not to choose.**

- **You want a tiny, no-Node, no-grammar tool.** Use
  [`files-to-prompt`](../files-to-prompt/) (Python) — it is the
  minimal version of this idea.
- **You only want one Python file's symbols, not a whole repo.**
  Use [`symbex`](../symbex/) — surgical, AST-correct, zero
  network.
- **You want the agent to *navigate* the repo itself.** That is
  what `aider`'s repo-map, `claude-code`'s `Grep`/`Glob`, and
  `opencode`'s file tools are for. Pre-packing with `repomix` is
  the wrong shape if the agent already has read access to the
  filesystem.
- **You are on an air-gapped machine** and the `--compress`
  grammars have not been pre-downloaded. The on-first-use grammar
  fetch will fail closed.
