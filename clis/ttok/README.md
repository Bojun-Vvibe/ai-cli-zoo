# ttok

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/ttok>
> Binary name: `ttok`

`ttok` is the **token counter and truncator** of the catalog. It is a
single-purpose Unix filter: text in (stdin or file path), token count
out — or, with `--truncate N`, text-truncated-to-N-tokens out. That
is the entire surface. There is no model client, no chat, no agent,
no config file.

The point is composition. Every other context-shaping CLI in this
catalog ([`files-to-prompt`](../files-to-prompt/),
[`symbex`](../symbex/), [`repomix`](../repomix/),
[`strip-tags`](../strip-tags/), [`marker`](../marker/)) emits text
whose token cost you can only measure *after* paying for the API
call — unless you pipe through `ttok` first. The two canonical
shapes are:

```sh
files-to-prompt src/ --cxml | ttok                       # how many tokens?
files-to-prompt src/ --cxml | ttok --truncate 100000 | llm '...'   # cap it
```

That makes `ttok` the cheap, deterministic gate between a packer and
a model client. `wc -c` overcounts in the BPE direction; `wc -w`
undercounts; `ttok` runs the actual `tiktoken` encoder so the number
matches what OpenAI / Anthropic / many other providers will bill.

## 1. Install footprint

- Pure Python, ~10 KB CLI plus the `tiktoken` Rust-backed encoder
  (~5 MB once vocabularies are downloaded). No network at runtime
  after vocab cache.
- `pipx install ttok` (recommended), `uv tool install ttok`, or
  `pip install ttok` inside a venv.
- Encoder selection via `-m / --model MODEL_NAME`: defaults to
  `gpt-4o`'s `o200k_base`, supports any name `tiktoken` knows
  (`gpt-4o`, `gpt-4`, `gpt-3.5-turbo`, `text-davinci-003`,
  raw encoding names like `cl100k_base` / `o200k_base` /
  `p50k_base` / `r50k_base`).
- First run of a previously-unseen vocab does one HTTPS download to
  OpenAI's public vocab CDN; subsequent runs read from the local
  cache and are fully offline. Set `TIKTOKEN_CACHE_DIR` to pin the
  cache for air-gapped pre-seeding.

## 2. License

Apache-2.0.

## 3. Models supported

**None as a client.** `ttok` does not call any LLM. The `-m` flag
selects a *tokenizer*, not a chat endpoint. For non-OpenAI models
whose tokenizer is not BPE-tiktoken (Claude's recent models,
Gemini, Llama-family), the count is an *approximation* — usually
within 10-20% of the provider's own count, which is good enough for
"will this fit in the context window" decisions and bad for
exact-cost forecasting. For exact Anthropic counts, pair with
`anthropic.Anthropic().messages.count_tokens(...)`; for Gemini, with
the `count_tokens` REST endpoint.

## 4. MCP support

None. `ttok` is a Unix filter — text in, integer or text out. If
you want an MCP-style "count tokens" tool, wrap it in a one-line
shell script and expose it from a custom MCP server.

## 5. Sub-agent model

None. One invocation tokenizes its input, prints the count (or the
truncated text), and exits. There is no loop.

## 6. Telemetry stance

**Off, with no opt-in.** The only network call ever made is the
one-time vocab download from `openaipublic.blob.core.windows.net`;
once cached, `ttok` is fully offline. No analytics ping, no
telemetry, no API key required.

## 7. Prompt-cache strategy

Not applicable — `ttok` does not call any model. The `tiktoken`
vocab files themselves are cached on disk after the first download,
so steady-state operation is local-only.

## 8. Hot keybinds

No TUI, no REPL. The flag set:

- `ttok hello world` — count tokens of literal arguments (`2`).
- `echo "hello world" | ttok` — count tokens of stdin.
- `ttok < big_file.txt` — count tokens of a file via shell
  redirection.
- `ttok -i README.md path/to/other.py` — count tokens across
  multiple files at once (single total).
- `ttok -m gpt-4o-mini hello world` — use a specific model's
  tokenizer.
- `ttok --truncate 1000 < big_file.txt` — emit the first 1000
  tokens of the file (text out, not count out). Useful as the last
  stage of a `pack | strip | truncate | llm` pipeline.
- `ttok --tokens hello world` — emit the integer token IDs (one
  per line), useful for debugging tokenization edge cases (where
  does the encoder split your terminology?).
- `ttok --as-tokens hello world` — emit each token as text on its
  own line, side-by-side with the IDs. Same debugging use case,
  human-readable.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Composability with every other catalog packer.
`ttok` is what makes `files-to-prompt`, `symbex`, `repomix`,
`strip-tags`, and `marker` *steerable* — without it, you discover
the token count by eating an API error, with it the cap is a `|
ttok --truncate N` away. The 30-second loop "did my packer just
emit 800k tokens? — `… | ttok`" is the single most common shape in
practice.

**Weakness.** Three:

1. **Counts are exact only for OpenAI tiktoken-vocab models.**
   Anthropic, Gemini, and Llama-family models use different
   tokenizers; `ttok`'s number is a useful upper-bound estimate,
   not a billing oracle, for those.
2. **Truncation is dumb.** `--truncate N` cuts at exactly N tokens,
   mid-sentence and mid-word. There is no "truncate at the nearest
   paragraph boundary" knob; for that, pre-process with
   [`strip-tags`](../strip-tags/) or split with `awk` before piping
   in.
3. **Single total over multiple files.** `ttok -i a.txt b.txt`
   prints one count, not per-file counts. For per-file accounting
   use a shell loop or look at [`repomix`](../repomix/) which prints
   per-directory token totals as part of its pack header.

**When to choose.**

- **You are about to run a million-token pipeline and want a free
  pre-flight check.** `… | ttok` answers "will this fit and what
  will it cost" before the API call.
- **You are building a deterministic prompt-budget gate** in CI.
  `[ "$(ttok < prompt.txt)" -lt 100000 ] || exit 1` is one line and
  has no LLM dependency.
- **You need to debug a tokenization mismatch** between what you
  thought the model would see and what it actually sees. `ttok
  --as-tokens` shows the BPE split character-by-character.
- **You are a [`llm`](../llm/) / [`mods`](../mods/) /
  [`shell-gpt`](../shell-gpt/) user** who wants to pre-trim a
  context-packer's output to a fixed token cap. `… | ttok
  --truncate 60000 | llm '...'` is the canonical shape.

**When not to choose.**

- **You need exact Anthropic / Gemini token counts.** Use those
  providers' own `count_tokens` endpoints; `ttok` will be in the
  ballpark but will not match the bill exactly.
- **You want a model-aware *cost* estimator** (tokens × per-token
  price × system overhead). `ttok` only does the tokens half;
  multiply yourself, or use `llm logs` to track real spend after
  the fact.
- **You want to truncate at semantic boundaries** (end of sentence,
  end of section). Pre-process with a real splitter; `ttok
  --truncate` is byte-aware-only at the token level.
- **You are not piping anything into an LLM CLI.** `ttok` only
  exists to make the rest of the pipeline measurable; in isolation
  it is a token counter and nothing more.
