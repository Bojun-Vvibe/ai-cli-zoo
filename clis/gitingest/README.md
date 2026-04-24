# gitingest

> Snapshot date: 2026-04. Upstream: <https://github.com/coderamp-labs/gitingest>
> Last verified version: **v0.3.1** (2025-07-31). License file:
> [`LICENSE`](https://github.com/coderamp-labs/gitingest/blob/main/LICENSE) —
> MIT.

A context packer with a single memorable trick: replace `hub` with
`ingest` in any GitHub URL — `github.com/owner/repo` becomes
`gitingest.com/owner/repo` — and you get a prompt-friendly,
single-text dump of the repo. The CLI is the same engine, run
locally on a path or a URL.

Like [`files-to-prompt`](../files-to-prompt/),
[`repomix`](../repomix/), [`code2prompt`](../code2prompt/), and
[`symbex`](../symbex/), it produces text *for* a downstream LLM
CLI; it does not call models itself. Its niche in that lineup is
**zero-friction remote ingestion** — a path, a URL, or a slug, all
through the same command — plus a hosted web UI for the same
engine that non-CLI users can paste into a chat.

## 1. Install footprint

- `pipx install gitingest` (recommended) or `pip install gitingest`.
- Pure Python, depends on `click`, `tiktoken`, `pathspec`, `tomli`,
  `httpx` for remote fetch.
- No daemon, no project-local state. The CLI writes a single
  `digest.txt` (default) per invocation.
- macOS, Linux, Windows. Also exposes a Python API
  (`from gitingest import ingest`) for embedding in scripts.
- Mirror of the engine runs at `gitingest.com` — same code, hosted.

## 2. License

MIT. License file at the repo root: `LICENSE`.

## 3. Models supported

**None.** Like the other packers in §5c, gitingest does not call
LLMs. It emits a text artifact (a "digest") that you pipe or paste
into any CLI in this catalog that does. Token counts in the
summary header use `tiktoken` (OpenAI's encoder) as an
approximation — accurate for OpenAI tiktoken-vocab models, an
upper-bound estimate for Anthropic / Gemini / Llama.

## 4. MCP support

**No.** No `--mcp` mode (unlike [`repomix`](../repomix/)). If you
want an MCP-aware agent to invoke a packer over stdio, use
`repomix --mcp` or `code2prompt --mcp`; gitingest is a one-shot
CLI.

## 5. Sub-agent model

**None.** One pass over the source, one digest out. No agent loop.

## 6. Telemetry stance

**Off** for the CLI. The hosted `gitingest.com` web service
obviously sees the URLs you submit and the rendered digests — that's
the service. The pip-installed CLI does not phone home, and remote
fetches go directly to `github.com` (or the host you specify), not
through `gitingest.com`.

## 7. Prompt-cache strategy

**None on the LLM side** (no LLM is called). On the *fetch* side,
remote clones are shallow (`--depth 1`) and not cached between
invocations by default — re-running on the same URL re-clones.
Pass `--source <local-path>` after a manual clone if you'll iterate
on the same repo and don't want to re-fetch.

## 8. Hot keybinds (REPL + subcommands)

No REPL — it's a one-shot CLI. The whole surface is one command,
`gitingest`:

| Invocation | Action |
|------------|--------|
| `gitingest <path>` | Pack a local directory into `digest.txt` |
| `gitingest <github-url>` | Shallow-clone and pack a remote repo |
| `gitingest owner/repo` | Slug shorthand for `https://github.com/owner/repo` |
| `-o <file>` / `--output <file>` | Write to a different file (`-` for stdout) |
| `-i '*.py,*.md'` / `--include-pattern` | Include-only glob patterns (comma-separated) |
| `-e '*.lock,*.svg'` / `--exclude-pattern` | Exclude glob patterns |
| `-s <bytes>` / `--max-size` | Skip files larger than N bytes |
| `-b <branch>` / `--branch <branch>` | Pick a non-default branch on remote |
| `--include-gitignored` | Override `.gitignore` filtering |
| `--token <gh-token>` | Auth for private repos (or `GITHUB_TOKEN` env) |

The default output shape is a header (file tree + token count) plus
a body of `=== file: path/to/file ===` blocks with the file
contents. Suitable for direct paste into a chat or for piping into
[`llm`](../llm/), [`mods`](../mods/), [`fabric`](../fabric/),
[`shell-gpt`](../shell-gpt/), etc.

## 9. Killer feature, weakness, when to choose

- **Killer:** **`gitingest <github-url>` works without a clone.**
  No other entry in §5c (`files-to-prompt`, `symbex`,
  `code2prompt`) fetches a remote repo as a first-class input —
  they all assume you have the source already on disk. `repomix`
  added `--remote owner/repo` for the same reason; gitingest's
  twist is that *the same engine has a hosted web UI* with the
  URL-renaming gimmick, so a teammate without the CLI can grab the
  same digest by editing a URL in their browser. That symmetry is
  unique in the catalog.
- **Weakness:** **no Tree-sitter compression, no Secretlint scan,
  no AST awareness.** `repomix` packs at the same level of
  granularity but ships compression and secret detection as
  built-ins; `symbex` is an order of magnitude smaller for
  Python-only symbol extraction; `code2prompt` lets you Handlebars-
  template the wrapper prompt. Gitingest is the most *minimal*
  packer — a glob filter, a max-size cap, and a clean text dump.
  When a digest is too big or contains secrets, you have to
  filter / scrub yourself.
- **Choose it when:** you want to ask a model about a repo you
  haven't cloned (`gitingest owner/repo | llm "what does this do?"`),
  or when a teammate needs to share a packed view of a repo via a
  web link rather than a CLI command. Use
  [`files-to-prompt`](../files-to-prompt/) instead for the most
  deterministic local subtree pack with `.gitignore` honoring;
  [`repomix`](../repomix/) for token-budget-aware compression and
  secret-scanning; [`code2prompt`](../code2prompt/) for templated
  export shapes and an interactive TUI; [`symbex`](../symbex/) for
  surgical Python-only symbol extraction; or
  [`marker`](../marker/) first if your "repo" is actually a stack
  of PDFs.

## Pitfalls observed in real use

1. **Remote fetch is unauthenticated by default.** Hits the GitHub
   public API, which has stricter rate limits than authenticated
   calls. On a busy network or for back-to-back `gitingest` calls,
   you'll see 403s. Set `GITHUB_TOKEN` in the environment (or pass
   `--token`) and the rate limit jumps from 60/h to 5000/h.
2. **`.gitignore` parsing is best-effort.** Nested `.gitignore`
   files are honored, but uncommon constructs (negation patterns
   re-including a previously ignored path, `/`-rooted patterns at
   non-root levels) occasionally diverge from `git check-ignore`'s
   behavior. If a file you expected to be excluded shows up in the
   digest, fall back to `-e <pattern>` rather than fighting the
   gitignore parser.
3. **`--max-size` is per-file, not cumulative.** A repo full of
   medium-sized files can still produce a multi-megabyte digest.
   Run `gitingest <path> | ttok` first
   (see [`ttok`](../ttok/)) to know the bill before paying it, or
   pipe the digest through `ttok --truncate <N>` to cap the
   downstream payload.
4. **Binary detection is heuristic.** Files with a non-text
   extension are skipped, but a text file in a binary-typical
   extension (`.dat` containing JSON, `.bin` containing a base64
   blob you do want) needs an explicit `-i` to come through. There
   is no `--force-text` flag — use the include pattern to opt
   specific extensions in.
5. **No secret scanning.** Unlike `repomix` (Secretlint-built-in),
   gitingest will happily emit `.env` files (if not gitignored),
   private keys checked into `tests/fixtures/`, and AWS access
   keys committed in old commits of the working tree. If you're
   piping digests to a hosted LLM, run `trufflehog` or
   `git-secrets` over the path *before* you ingest, or switch to
   `repomix` which catches the common cases natively.
6. **Hosted `gitingest.com` and the local CLI are the same engine
   but not the same trust boundary.** A teammate URL-renaming
   `github.com/...` to `gitingest.com/...` is sending the rendered
   digest through a third-party host. For private repos or
   compliance-sensitive code, insist on the local CLI and don't
   share the `.com` link.
