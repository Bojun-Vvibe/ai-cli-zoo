# marker

> PDF / EPUB / DOCX / PPTX → clean Markdown for LLM ingest. As of the
> 1.x line (late 2025).

## TL;DR

`marker` is a CLI (and Python library) that converts documents — PDFs
especially, including scanned and multi-column academic papers — into
LLM-friendly Markdown with extracted tables, equations (LaTeX), code
blocks, and image references. It runs a pipeline of layout detection,
OCR (Surya), table recognition, and equation parsing, optionally with
an `--use_llm` finishing pass that hands ambiguous regions
(merged-cell tables, hand-drawn diagrams, broken inline math) to a
real LLM for repair. Output is a directory with `*.md`, extracted
images, and a `meta.json` (page-by-page block tree, language guesses,
confidence scores).

This is a *converter*, not a chat tool. It exists to feed every other
CLI in this catalog.

## Install

```bash
pip install marker-pdf            # CPU works; CUDA / MPS auto-detected
# heavy deps: PyTorch, Surya OCR weights (~2GB on first run, cached)
```

CLI entrypoints:

```
marker_single  input.pdf  output_dir/
marker         input_dir/ output_dir/   # batch
marker_server                            # FastAPI server, /convert
marker_chat                              # streamlit demo UI
```

## One Concrete Example

```
$ marker_single paper.pdf out/ --output_format markdown --use_llm \
    --llm_service marker.services.openai.OpenAIService

Loading layout model: surya/layout (cached)
Loading OCR model: surya/ocr (cached)
Loading table model: surya/table_rec (cached)
Loaded models in 4.1s.
Detected 14 pages, 2-column layout, 6 tables, 31 equations.
LLM repair pass: 6 tables, 4 ambiguous equations.
Wrote out/paper/paper.md (38 KB), 12 images, meta.json.
Done in 41s on Apple M2 (MPS).

$ head -20 out/paper/paper.md
# Attention Is All You Need

## Abstract
The dominant sequence transduction models …

| Model              | BLEU  | Training cost (FLOPs) |
|--------------------|-------|------------------------|
| ByteNet            | 23.75 | …                      |
…

$$
\mathrm{Attention}(Q, K, V) = \mathrm{softmax}\!\left(\frac{QK^\top}{\sqrt{d_k}}\right) V
$$
```

Pipe straight into a packer / LLM:

```bash
marker_single rfc.pdf rfc-md/ && \
  cat rfc-md/rfc/rfc.md | llm -m claude-sonnet 'extract the MUST clauses as a checklist'
```

## Niche It Fills

**Document → Markdown so other CLIs can do their job.** None of the
catalog's chat / agent CLIs ingest PDFs natively in any useful way —
they either drop a base64 image into the context (expensive, lossy)
or refuse. `marker` produces deterministic, table- and
equation-preserving Markdown, which is the format every other CLI in
this repo already eats. It is the upstream half of "chat with this
paper / spec / contract / scanned manual."

## Vs Already Cataloged

- **Vs [`files-to-prompt`](../files-to-prompt/) / [`repomix`](../repomix/):**
  Those are *source-tree* packers. They have nothing to say about
  PDFs. `marker` is the missing front-end: PDF → MD → packer → LLM.
- **Vs [`aichat`](../aichat/) RAG mode:** `aichat` will index a
  folder of *text* files for retrieval. Feed it a folder of raw PDFs
  and you get garbage. Run `marker` first, then point `aichat .rag
  add` at the resulting `.md` tree.
- **Vs ad-hoc `pdftotext`:** `pdftotext` loses tables, columns, and
  math. For a one-page receipt that is fine. For a 40-page paper
  with two-column layout and six tables it is unusable as LLM input;
  `marker` is purpose-built for exactly that case.

## Caveats

- First run downloads ~2GB of model weights to `~/.cache/`. Plan
  accordingly on locked-down boxes.
- CPU-only is *slow* (minutes per academic paper). MPS / CUDA is
  ~5-10× faster. Batch mode (`marker input_dir/`) parallelizes by
  page across workers — a 100-doc corpus on a laptop is a coffee
  break, not a session.
- `--use_llm` repair sends excerpts of your document to whichever
  provider you configure. If the docs are confidential, leave it off
  (the non-LLM pipeline still produces usable Markdown; you just
  lose some table-merge and equation-edge-case quality).
- Equation extraction is good but not perfect — verify any LaTeX
  you plan to *render* downstream.
- AGPL-style restrictions apply for some commercial use; check the
  license before bundling into a hosted product.
