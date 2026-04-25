# docling

> Snapshot date: 2026-04. Upstream: <https://github.com/docling-project/docling>
> License file: <https://github.com/docling-project/docling/blob/main/LICENSE>
> Pinned: `v2.91.0` (2026-04-23). Linux Foundation AI & Data project
> (donated by IBM Research, ongoing development across multiple
> contributors). `main` ships frequently; pin the exact PyPI version
> in any RAG pipeline because the layout / table models change between
> minor releases and chunking output shifts with them.

A **document → unified `DoclingDocument` IR → many export formats**
converter. Reads PDFs, DOCX, PPTX, XLSX, HTML, images (PNG/TIFF/JPEG),
LaTeX, plain text, WebVTT, MP3/WAV (via ASR) and a few application
schemas (USPTO patents, JATS articles, XBRL financials). Emits
Markdown, HTML, lossless JSON, WebVTT, or `DocTags` (the project's
own structured tag dialect). Same problem space as
[`marker`](../marker/) and [`zerox`](../zerox/), with a different
philosophy: **local ML pipeline by default, no remote LLM in the
critical path**, with optional VLM (`GraniteDocling`, others) for the
hard pages.

The CLI is plain `docling`; one-liner for the common case is
`docling input.pdf` (writes Markdown next to the input).

## 1. Install footprint

- `pip install docling` (or `uv pip install docling`). Python 3.10+.
  First run downloads layout / table / OCR model weights (~1–2 GB)
  to a Hugging Face cache; budget for that on a fresh box.
- Works on macOS, Linux, Windows; x86_64 and arm64. On Apple Silicon
  the `--pipeline vlm` mode lights up MLX acceleration automatically
  if `mlx` is installed.
- No daemon, no project state. Output is a file (or stdout); model
  weights live in `~/.cache/huggingface/`. The MCP server (`docling
  mcp`) is opt-in and runs in your terminal foreground.
- Workspace footprint per run: nothing on disk beyond the chosen
  output file. Intermediate page images and layout overlays stay in
  RAM unless you pass `--debug-layout-output ./out`.

## 2. License

MIT.

## 3. Models supported

Two distinct paths:

- **Default pipeline (no LLM)** — local Heron layout model + table
  structure model + EasyOCR / Tesseract / RapidOCR for scanned text +
  classical readers for born-digital PDFs. Zero network egress, zero
  API keys, deterministic per version.
- **VLM pipeline (`--pipeline vlm`)** — passes whole pages to a
  vision-language model:
  - **GraniteDocling** (`granite_docling`) — a 258M-param VLM tuned
    by the project for document layout; runs locally via Transformers
    or MLX.
  - **Any other VLM** wired through the `--vlm-model` flag —
    SmolDocling, Qwen2-VL, etc., picked up from HF.
  - Optional **remote VLMs** (OpenAI vision, Anthropic vision) via
    Python API only, not the default CLI.

For embedding into a downstream RAG, docling itself does no
embedding; you hand the `DoclingDocument` (or its Markdown export) to
LangChain / LlamaIndex / Haystack — first-class integrations exist
for all three.

## 4. MCP support

**Yes — server.** `docling mcp` (the docs cover the full setup) starts
a local MCP server that exposes document conversion as a tool. Hook
it into [`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`goose`](../goose/), or any other MCP client and you get
"convert this PDF for me" as a first-class agent capability — no
shelling out, no temp files in agent prompts.

## 5. Sub-agent model

None — docling is a converter, not an agent. The pipeline is
deterministic stages (parse → layout → reading-order → tables → OCR
→ assemble → export). The optional VLM mode is one model call per
page with no review pass.

## 6. Telemetry stance

**Off, no opt-in.** No analytics, no phone-home. The default pipeline
is fully offline once weights are cached. The VLM pipeline only sends
data anywhere if you explicitly point `--vlm-model` at a remote
endpoint. The MCP server binds locally and only talks to the MCP
client you connect.

## 7. Prompt-cache strategy

N/A for the default pipeline (no LLM). For the VLM pipeline there is
no caching layer; each page is a fresh model call. The reusable cost
saver is the **lossless JSON export**: convert once, dump
`DoclingDocument` as JSON, then re-derive Markdown / HTML / chunks
from the JSON without re-running the heavy ML pipeline.

## 8. Hot keybinds

No TUI. CLI surface (selected):

- `docling input.pdf` — convert, write Markdown next to the source.
- `docling input.pdf --to html --to json` — multi-format export in
  one pass.
- `docling input.pdf --output ./out` — write into a directory.
- `docling https://arxiv.org/pdf/2206.01062` — accepts URLs directly.
- `docling --pipeline vlm --vlm-model granite_docling input.pdf` —
  use the VLM pipeline (slower, better on hard pages, MLX-accelerated
  on Apple Silicon).
- `docling --ocr-engine easyocr input.pdf` — pick the OCR backend
  (`easyocr`, `tesseract`, `rapidocr`, `ocrmac` on macOS).
- `docling --no-ocr input.pdf` — skip OCR for speed on born-digital
  PDFs.
- `docling --image-export-mode referenced input.pdf` — extract
  embedded images alongside the Markdown (vs `placeholder` /
  `embedded`).
- `docling --enrich-formula --enrich-code input.pdf` — turn on the
  optional formula / code-block enrichment models.
- `docling extract input.pdf --schema schema.json` — beta structured
  extraction (information extraction with a JSON schema).
- `docling mcp` — start the MCP server for agentic clients.

Programmatic:

```python
from docling.document_converter import DocumentConverter
result = DocumentConverter().convert("https://arxiv.org/pdf/2408.09869")
print(result.document.export_to_markdown())
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The unified `DoclingDocument` IR.** Other
converters in this catalog ([`marker`](../marker/),
[`zerox`](../zerox/)) emit Markdown directly and you take what you
get. docling preserves a typed tree — pages, blocks, tables (cells
with row/col spans), figures (with bounding boxes), formulas (LaTeX),
code blocks, headings, list nesting, reading order — and then has
exporters on top of that. That means you can convert once, ship the
JSON, and re-render to Markdown / HTML / chunks tuned for your RAG,
without re-running the expensive layout pass. It is also the only
catalog entry that handles **PDF + DOCX + PPTX + XLSX + audio
(via ASR) + XBRL/JATS schemas** behind one CLI, so a heterogeneous
corpus becomes one ingestion command. And the **MCP server** makes
"convert document X" a tool any MCP-aware agent can call directly.

**Weakness.** **Heavy install** — first run pulls 1–2 GB of model
weights to the HF cache. **Slower than `pdftotext` and `marker`** on
clean born-digital PDFs because the layout model runs by default
(use `--no-ocr` and the default pipeline is competitive again).
The **VLM pipeline is GraniteDocling-flavoured** out of the box —
quality is good for a 258M model but well below GPT-4o or Claude
Sonnet on adversarial scans, so for "must-not-fail" extraction from
messy documents [`zerox`](../zerox/) with a frontier vision model
still wins on quality (and loses on cost, latency, and data
residency). Structured extraction (`docling extract`) is still
flagged beta in the README.

**When to choose.**

- You need to **ingest a heterogeneous document corpus** (mixed PDF +
  DOCX + PPTX + audio + XBRL) and want one CLI / one IR rather than
  three pipelines.
- You want **clean Markdown for a RAG pipeline** with preserved
  tables, formulas, and reading order, **without sending pages to a
  cloud LLM**.
- You want to expose document conversion as a **first-class MCP tool**
  to an agent.
- Your environment is **air-gapped or regulated** and the LLM-OCR
  approach in [`zerox`](../zerox/) is a non-starter.

**When not to choose.** You only have clean born-digital PDFs and you
want speed — `pdftotext` (poppler) is a single shell command and an
order of magnitude faster. You have a small number of adversarial
scans (handwriting, multi-column with merged-cell tables) and quality
is paramount — [`zerox`](../zerox/) with GPT-4o or Claude 3.7 Sonnet
will beat docling-VLM. You do not want to manage a 1–2 GB model
cache — [`marker`](../marker/) is heavier still, but `pdftotext`
ships in poppler.
