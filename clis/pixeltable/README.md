# pixeltable

> Snapshot date: 2026-04. Upstream: <https://github.com/pixeltable/pixeltable>

"**AI data infrastructure providing a declarative, incremental
approach for multimodal workloads.**" Pixeltable is a Python-native
table store that treats *any* media type — text, image, audio,
video, documents — as a first-class column, and lets you declare
*computed columns* whose values are LLM / embedding / vision-model
outputs. The runtime handles incremental recomputation, automatic
caching, and on-disk versioning, so you write what you want
(`t.add_computed_column(caption=openai_chat([...]))`) and Pixeltable
figures out what needs to run when new rows land. Vector search,
multimodal RAG, video frame extraction, and agent memory all
collapse into "add a column."

## 1. Install footprint

- `pip install pixeltable` (core, includes embedded Postgres + the
  storage layer; no external DB to set up).
- Optional model integrations are extras-free imports:
  `pixeltable.functions.openai`,
  `pixeltable.functions.anthropic`,
  `pixeltable.functions.huggingface`,
  `pixeltable.functions.whisperx`,
  `pixeltable.functions.yolox`,
  `pixeltable.functions.sentence_transformers`,
  `pixeltable.functions.fireworks`, `mistralai`, `together`,
  `replicate`, `gemini`, `groq`, `deepseek`, `bedrock`.
- Python ≥ 3.10. macOS, Linux, Windows. Bundles its own embedded
  Postgres binary — no separate database to install.
- No CLI binary; the surface is the Python API plus a Streamlit /
  notebook UX. (`pxt` shell is provided via `python -m pixeltable`.)

## 2. Repo + version + license

- Repo: <https://github.com/pixeltable/pixeltable>
- Latest release: **v0.5.28** (2026-04-17)
- License: **Apache-2.0** —
  <https://github.com/pixeltable/pixeltable/blob/main/LICENSE>
- HEAD SHA: `c59468f644b5e72a9cd34973a0138dd93fc58996`
- Default branch: `main`
- Language: Python (Cython hot path for the storage layer)

## 3. Models supported

LLMs: OpenAI, Anthropic, Mistral, Together, Fireworks, Groq,
Replicate, DeepSeek, Gemini, Bedrock, Ollama, llama.cpp, vLLM via
OpenAI-compat. Embeddings: OpenAI, Cohere,
sentence-transformers, CLIP, BGE, any HF model. Vision: YOLOX,
Florence-2, CLIP, custom HF pipelines. Audio: WhisperX. Video:
ffmpeg + frame iterators feed the same column model. Custom UDFs
via the `@pxt.udf` decorator wrap any Python callable into a
typed computed-column function.

## 4. Simple usage

```python
import pixeltable as pxt
from pixeltable.functions.openai import chat_completions, embeddings

pxt.create_dir("docs", if_exists="ignore")

# table with an Image column — Pixeltable stores files by reference
t = pxt.create_table(
    "docs.papers",
    {"title": pxt.String, "pdf": pxt.Document, "img": pxt.Image},
    if_exists="ignore",
)

# computed column: caption every image with GPT-4o-mini
t.add_computed_column(
    caption=chat_completions(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": [
            {"type": "text", "text": "Describe this image."},
            {"type": "image_url", "image_url": {"url": t.img}},
        ]}],
    ).choices[0].message.content
)

# vector index over the captions
t.add_embedding_index("caption", string_embed=embeddings.using(model="text-embedding-3-small"))

# semantic search returns the original rows + computed cols + media refs
hits = t.where(
    pxt.functions.string.similarity(t.caption, "graph neural network for molecules") > 0.5
).select(t.title, t.img, t.caption).limit(5).collect()
```

```python
# video frames as a virtual column — extract, embed, search
v = pxt.create_table("docs.videos", {"clip": pxt.Video})
frames = pxt.create_view("docs.frames", v.tbl_iterator(pxt.iterators.FrameIterator(v.clip, fps=1)))
frames.add_embedding_index("frame", image_embed=embeddings.using(model="clip-ViT-B-32"))
# now you can search "person riding a bicycle" across hours of video
```

## 5. Why it's interesting

- **Computed columns are the API.** You don't write a pipeline
  or a DAG — you declare "this column = this function(other
  columns)" and the runtime decides when to compute, cache, and
  invalidate. Re-running on a new model = swap the function and
  the rows recompute incrementally.
- **Multimodal first-class.** `Image` / `Video` / `Audio` /
  `Document` are real column types with bytewise dedup, lazy
  decoding, and frame iterators — not "VARCHAR with a path
  string."
- **Embedded Postgres + on-disk media** means the whole stack
  ships with `pip install`; no Docker, no managed vector DB to
  provision before you can index.
- **Versioning is built in.** Every table mutation is versioned;
  `t.revert()` rolls back; computed-column re-applies after
  schema change are automatic.

## 6. Caveats

- The embedded-Postgres deployment model is great for laptops
  and single-node prototypes; multi-node sharding is not yet a
  supported topology.
- API surface is still 0.x — column-decorator signatures and
  `add_computed_column` ergonomics have changed across recent
  minor versions; pin a version.
- Computed-column re-runs on a new model can be expensive on
  large media tables; use `if_exists="ignore"` and
  `--max-rows` patterns when iterating model choices.
