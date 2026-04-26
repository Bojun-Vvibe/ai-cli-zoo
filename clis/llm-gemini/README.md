# llm-gemini

> Snapshot date: 2026-04-26. Upstream: <https://github.com/simonw/llm-gemini>

"**LLM plugin to access Google's Gemini family of models.**"
`llm-gemini` is Simon Willison's first-party plugin for the
[`llm`](../llm/) CLI that wires Google AI Studio's Gemini chat,
reasoning, embedding, and image-generation endpoints into the
same `llm "..."`/`llm chat`/`llm embed` surface every other
plugin in the catalog uses — so a single `~/.config/io.datasette.llm/`
SQLite log, the same prompt-templating, and the same
`-c`/conversation continuity now span Gemini alongside whatever
OpenAI / Anthropic / Mistral models the user already has.

## 1. Install footprint

- Plugin install: `llm install llm-gemini` (requires
  [`llm`](../llm/) ≥ 0.26 already on PATH).
- Auth: `llm keys set gemini` then paste an AI Studio key, or
  export `LLM_GEMINI_KEY` / `GEMINI_API_KEY`.
- Use: `llm -m gemini-2.5-pro 'summarize this PDF' -a paper.pdf`,
  `llm -m gemini-2.5-flash chat`, `llm embed -m gemini-embedding
  -c "..."`, or `llm -m gemini-2.5-flash-image-preview 'draw a
  pelican on a bicycle' -o image_output_path out.png`.
- No standalone binary — lives entirely as Python inside the
  parent `llm` venv; uninstall is `llm uninstall llm-gemini`.

## 2. Repo + version + license

- Repo: <https://github.com/simonw/llm-gemini>
- Latest release: **0.30** (published 2026-04-02)
- License: **Apache-2.0** —
  <https://github.com/simonw/llm-gemini/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Stars: ~440

## 3. Models supported

- Chat / reasoning: `gemini-2.5-pro`, `gemini-2.5-flash`,
  `gemini-2.5-flash-lite`, plus the `-preview-MM-DD` snapshots
  Google ships weekly; each exposes `-o thinking_budget N` for
  the 2.5 family's controllable reasoning tokens.
- Multimodal in: PDF, image, audio, video attachments via
  `llm -a path/url` flow inherited from the parent CLI.
- Multimodal out: image generation via
  `gemini-2.5-flash-image-preview` (the "Nano Banana" model)
  with `-o image_output_path` writing PNG to disk.
- Embeddings: `gemini-embedding-001` and `text-embedding-004`
  reachable via `llm embed-models` and usable as the vector
  source for `llm similar` / `llm cluster`.
- Tool calling + structured output via `llm`'s `--functions`
  and `--schema` flags; Google Search grounding via
  `-o google_search 1`; URL context via `-o url_context 1`.

## 4. Notable angle

**It is the lowest-friction way to put Gemini into a workflow
that already revolves around Simon Willison's `llm` CLI.** Where
[`google-generativeai`](https://pypi.org/project/google-generativeai/)
the SDK gives you a Python client and nothing else, and where
the Vertex AI SDK gives you GCP IAM + regional endpoints and
nothing else, `llm-gemini` gives you Gemini *as a peer* to every
other provider in the catalog: the prompts are logged to the
same SQLite database [`llm`](../llm/) uses for OpenAI / Anthropic
/ Ollama runs, conversations continue across providers with
`-c`, templates compose with `--system`/`--schema`/`--functions`
the same way, and the cost of switching from `-m gpt-4o` to
`-m gemini-2.5-pro` is one flag. The trade-off is the inverse:
you inherit `llm`'s opinions (SQLite log, no streaming UI beyond
what `llm chat` ships, no agent loop — for that, point the
[`llm-cmd`](../llm-cmd/) or [`llm-functions`](../llm-functions/)
plugins at the same model). Use it when Gemini is one of several
providers in your shell-side toolbox; reach for the official
Google SDK when you need Vertex IAM, regional pinning, or
batch / tuning APIs that AI Studio doesn't expose.

## 5. Last verified

2026-04-26 via `gh api repos/simonw/llm-gemini` and
`gh api repos/simonw/llm-gemini/releases/latest` → 0.30
(2026-04-02), license Apache-2.0 at
<https://github.com/simonw/llm-gemini/blob/main/LICENSE>.
