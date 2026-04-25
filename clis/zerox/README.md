# zerox

> Snapshot date: 2026-04. Upstream: <https://github.com/getomni-ai/zerox>
> License file: <https://github.com/getomni-ai/zerox/blob/main/LICENSE>
> Pinned: `v0.1.06` (2024-12-18). The repo continues to receive commits
> on `main` after the last tagged release; pin a commit SHA in CI rather
> than tracking `latest`.

A CLI/library that turns **PDFs, DOCX, XLSX, images, and other documents
into clean Markdown** by rendering each page to an image and asking a
**vision-capable LLM** to transcribe it. Zero traditional OCR, zero
layout heuristics — the model is the OCR. Same output goal as
[`marker`](../marker/), opposite philosophy: marker uses local ML +
heuristics, zerox uses a remote vision model.

There is no installable binary; it runs as `npx zerox <file>` (Node) or
`from pyzerox import zerox` (Python).

## 1. Install footprint

- **Node:** `npx zerox path/to/file.pdf` (no install) or
  `npm i -g zerox` for a persistent global. Requires Node 18+ and a
  local `graphicsmagick` + `ghostscript` install for PDF rasterisation
  (`brew install graphicsmagick ghostscript` on macOS).
- **Python:** `pip install py-zerox` (note the dash). Pulls in
  `pdf2image` which itself wants `poppler` on the system.
- No daemon, no project state. Output is whatever directory you point
  `--output-dir` at; intermediate page images live in a tempdir.
- Workspace footprint per run: one PNG per page (deleted on exit
  unless you pass `--cleanup false`).

## 2. License

MIT.

## 3. Models supported

Anything `litellm` can route to **that has vision** — in practice:

- **OpenAI** — `gpt-4o`, `gpt-4o-mini`, `gpt-4-vision-preview`
- **Anthropic** — Claude 3.5 / 3.7 Sonnet, Haiku, Opus (all
  vision-enabled)
- **Gemini** — `gemini-1.5-pro`, `gemini-2.0-flash`
- **AWS Bedrock** — Claude vision models
- **Azure OpenAI** — vision deployments
- **Local** — any Ollama / vLLM model with image input
  (`llava`, `llama3.2-vision`, `qwen2-vl`, etc.) via the
  OpenAI-compatible endpoint

Choice of model is the single biggest quality knob: GPT-4o and Claude
3.7 Sonnet handle complex tables and handwriting; smaller models
struggle. The cost per page is also model-dominated — budget
$0.005–$0.02 per page on frontier models, ~$0.0005 on Haiku.

## 4. MCP support

**No.** zerox is a one-shot document → Markdown converter; it has no
need for the MCP surface and exposes none. Wrap it in your own MCP
server if you want an agent to call it.

## 5. Sub-agent model

None. Each page is one independent vision call; the tool runs them
concurrently (`--concurrency N`, default 10) but there is no planning
loop, no review pass, no second-model verification. If the model
hallucinates a number in a table, that lands in your Markdown.

## 6. Telemetry stance

**Off, with no opt-in.** No analytics SDK in either the Node or
Python package. The vision model provider obviously sees every page
image you send (which is the entire payload — be aware of this for
sensitive documents). The tool itself does not phone home.

## 7. Prompt-cache strategy

None. Each page is a fresh vision call with a fresh image — vision
inputs are not eligible for prefix caches on any current provider, and
the tool does not attempt it. The only "cache" is the on-disk output:
a re-run with the same source file and same `--output-dir` will skip
already-converted pages if you pass `--maintain-format`.

## 8. Hot keybinds

No TUI. Node CLI surface:

- `npx zerox doc.pdf` — convert, write Markdown next to the source
- `npx zerox doc.pdf -m gpt-4o-mini` — pick the vision model
- `npx zerox doc.pdf -o ./out` — write Markdown + images to `./out`
- `npx zerox doc.pdf --concurrency 20` — parallel page calls
- `npx zerox doc.pdf --pages 1-10` — page range
- `npx zerox doc.pdf --maintain-format` — feed previous page's
  Markdown into the next page's prompt as a style anchor (improves
  table / heading consistency across long documents)
- `npx zerox doc.pdf --schema schema.json` — structured extraction
  mode; emits JSON matching the schema instead of free-form Markdown
- `npx zerox https://example.com/file.pdf` — accepts URLs directly
- `npx zerox doc.pdf --cleanup false` — keep the rasterised page PNGs
  for inspection

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Vision-model OCR is the right answer for messy
documents**, and zerox is the smallest possible wrapper around that
idea. Scanned forms with handwritten annotations, multi-column
academic PDFs with formulas, financial statements with merged-cell
tables, screenshots of dashboards — these are the cases where
classical OCR (Tesseract, Apache Tika) and ML-pipeline tools (marker)
either lose structure or hallucinate it. A frontier vision model
hands you correct Markdown on the first try because it understands
the *layout intent*, not just the pixels. zerox lets you cash that in
with one CLI invocation and no infrastructure.

**Weakness.** **Cost** (vision tokens are expensive — a 200-page PDF
on GPT-4o is real money), **latency** (~3–8 seconds per page even with
concurrency), and **non-determinism** (the model will occasionally
mis-read a digit; there is no review pass). Also: every page image
leaves your machine and goes to the model provider. For regulated
documents, run against a self-hosted vision model (`llava` /
`llama3.2-vision` via Ollama) — quality is meaningfully worse but the
data stays local.

**When to choose.**

- You have **complex documents** (multi-column, tables, handwriting,
  forms, scans) and you have already tried `pdftotext` /
  [`marker`](../marker/) and the output is not usable.
- You need a **one-shot converter** with no Python ML stack to
  install and maintain.
- You want **structured extraction with a JSON schema** from
  document images.
- You are building a **RAG pipeline over PDFs** and want the
  cleanest possible Markdown to feed to the chunker.

**When not to choose.** You have thousands of clean, simple PDFs —
[`marker`](../marker/) or even `pdftotext` will be 100× cheaper and
fast enough. Your documents contain regulated data and you have no
self-hosted vision model — pick a local OCR pipeline. You need
deterministic, auditable extraction (legal, accounting) — vision
models will sometimes "tidy up" what they see, which is precisely
what those workflows cannot tolerate.
