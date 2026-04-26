# inference-roboflow

> Snapshot date: 2026-04-26. Upstream: <https://github.com/roboflow/inference>

"**Turn any computer or edge device into a command center for
your computer vision projects.**" Roboflow `inference` is a
self-hosted vision server + Python SDK + `inference` CLI that
exposes hundreds of pre-trained CV models (YOLO v5/v7/v8/v9/v10/
v11, RF-DETR, SAM 2, CLIP, Florence-2, PaliGemma, Qwen2.5-VL,
DocTR OCR, etc.) and a "Workflows" no-code DAG builder behind
one HTTP endpoint, so a `POST /infer/object_detection`
returns predictions whether the workload is on a Jetson, a
Raspberry Pi, a laptop CPU, or a fleet of T4s — same API,
same JSON shape.

## 1. Install footprint

- Python: `pip install inference` (CPU) / `inference-gpu`
  (CUDA) / `inference-cli`; Apple-silicon and Jetson wheels
  are first-class.
- CLI: `inference server start` boots the local FastAPI server
  on `:9001` (Docker under the hood — pulls
  `roboflow/roboflow-inference-server-cpu` or `-gpu` /
  `-jetson-*` per platform); `inference benchmark python-
  package-speed -m yolov8n-640` measures device throughput;
  `inference workflows run --workspace-name X --workflow-id Y`
  executes a Workflow against a video / RTSP / image batch.
- Bring-your-own-weights via Roboflow Universe model IDs
  (`model_id="yolov8n-640"` etc.) cached under
  `~/.inference/cache/`; runs fully offline once weights land.

## 2. Repo + version + license

- Repo: <https://github.com/roboflow/inference>
- Latest release: **v1.2.5** (published 2026-04-24)
- License: **Apache-2.0** for `LICENSE.core` (everything
  outside `inference/enterprise/` and per-model carve-outs in
  `inference/models/`); see top-level `LICENSE` for the
  carve-out map —
  <https://github.com/roboflow/inference/blob/main/LICENSE>
  and <https://github.com/roboflow/inference/blob/main/LICENSE.core>
- Default branch: `main`
- Language: Python
- Stars: ~2.3k

## 3. Models supported

- Object detection: YOLO v5 / v7 / v8 / v9 / v10 / v11
  (n/s/m/l/x), YOLO-NAS, YOLO-World (open-vocabulary),
  RF-DETR, Roboflow 3.0.
- Segmentation: YOLO-seg variants, SAM 2 (image + video),
  Mask R-CNN.
- Keypoint / pose: YOLO-pose v8/v11.
- Classification: YOLO-cls, ViT, ResNet.
- VLMs / foundation: CLIP (zero-shot embed + classify),
  Florence-2, PaliGemma, Qwen2.5-VL, SmolVLM, Moondream2,
  Grounding DINO, OWL-ViT.
- OCR: DocTR, TrOCR, EasyOCR adapters.
- Depth + tracking: Depth Anything v2; ByteTrack / BoT-SORT
  trackers wired into Workflows.
- Custom Roboflow-trained models pulled by `model_id` from
  Roboflow Universe / private workspace.

## 4. Notable angle

**It is the only entry in the catalog that treats "production
computer vision" as a first-class deployable service rather
than a notebook script.** Where [`ollama`](../ollama/) made
"one HTTP server, dozens of LLMs, runs on my laptop" the
default for text models, `inference` is the same shape for
vision: one Docker image per device class (CPU / CUDA / Jetson
Orin / Jetson Nano / RPi), one stable HTTP API
(`/infer/object_detection`, `/infer/instance_segmentation`,
`/infer/workflows/...`), and a Workflows DSL that turns "detect
people → crop → classify PPE → push alert to webhook" into a
JSON DAG you can ship to a Jetson without writing per-device
code. The pairing with the [`supervision`](https://github.com/roboflow/supervision)
annotator library means the same predictions visualize the
same way in a notebook, in `inference benchmark` output, and
in a deployed dashboard. The trade-off: it is vision-only (no
LLM hosting — pair with [`ollama`](../ollama/) /
[`vllm`](../vllm/) for text), the Workflows visual builder
lives on roboflow.com (the runtime is OSS but the no-code
authoring UI is the hosted service), and the
`inference/enterprise/` directory and select model weights
have non-Apache licenses you must read before embedding in a
commercial product. Use it when you need a real CV service on
edge hardware with a stable API; skip it when a one-off
notebook with raw Ultralytics or a Hugging Face pipeline is
all the workload demands.

## 5. Last verified

2026-04-26 via `gh api repos/roboflow/inference` and
`gh api repos/roboflow/inference/releases/latest` → v1.2.5
(2026-04-24), Apache-2.0 (with carve-outs) at
<https://github.com/roboflow/inference/blob/main/LICENSE.core>.
