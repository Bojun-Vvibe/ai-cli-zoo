# fiftyone

- **Repo:** https://github.com/voxel51/fiftyone
- **Version:** v1.14.2 (released 2026-04-23; HEAD pinned 2fbca66, branch `develop`)
- **License:** Apache-2.0 (`LICENSE`, blob SHA `a598f5e6c368fb2d36ec5cd00639551c77bf3735`)
- **Category:** Dataset curation + visual-AI exploration

## What it does

A Python library + `fiftyone` CLI + browser-based App for **building,
inspecting, and curating computer-vision / multimodal datasets** —
images, video, 3D point clouds, with object detections, segmentations,
classifications, embeddings, and arbitrary per-sample metadata as
typed fields. The App renders a sample grid, a per-sample modal view
with predictions overlaid on pixels, an embeddings projector
(`fob.compute_visualization` → UMAP / t-SNE), and a "similarity"
search backed by a configurable vector index (sklearn, Qdrant, Pinecone,
LanceDB, Milvus, Redis, MongoDB Atlas Vector Search) so finding the
30 hardest validation examples for a model is a UI gesture rather than
a notebook detour.

## Install

```sh
pip install fiftyone
fiftyone app launch                 # open the App at :5151
fiftyone datasets list              # show local datasets
fiftyone zoo datasets load coco-2017-validation
fiftyone plugins download <url>     # install a plugin
```

## Why it's interesting

The dataset-side counterpart to model-evaluation harnesses: where
[`promptfoo`](../promptfoo/) and [`deepeval`](../deepeval/) score model
outputs, fiftyone makes the **inputs and labels** queryable as a typed
collection (`dataset.match(F("predictions.detections").length() == 0)`
returns the false-negatives) so error analysis stops being grep over
JSON. The Brain layer (`fiftyone.brain`) packages the standard set of
"what's wrong with my dataset" routines as one-liners — uniqueness
sampling, near-duplicate detection, mistakenness scoring, hardness,
representativeness, embedding visualisation — each writing scores back
as fields the App can sort by. Plugin system + JS panels make it the
catalog's most extensible visual-AI workbench, and the v1 release
formalised a Python SDK + REST API stable enough that an agent CLI
can drive it as a long-running tool.
