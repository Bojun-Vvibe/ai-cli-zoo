# helm

> Snapshot date: 2026-04. Upstream: <https://github.com/stanford-crfm/helm>

**Stanford CRFM's holistic LLM evaluation framework.**
HELM (*Holistic Evaluation of Language Models*) is the
benchmark-of-benchmarks behind the Stanford CRFM leaderboards
(HELM Lite, HELM Classic, HELM Instruct, HELM MMLU, HELM Safety,
HELM Capabilities, HELM Robustness, HELM-VLM, MedHELM, FinanceBench,
EnterpriseBench). A `RunSpec` selects a `Scenario` (the data /
task), an `Adapter` (how examples become prompts), and a list of
`Metric`s (accuracy, calibration, robustness, fairness, bias,
toxicity, efficiency); `helm-run` evaluates models against the
spec and `helm-summarize` + `helm-server` produce the public-facing
HTML leaderboard pages.

## Repo + version + license

- Repo: <https://github.com/stanford-crfm/helm>
- Latest release: **`v0.5.15`** (2026-04-23)
- HEAD on `main`: `1193709` (2026-04-23)
- License: **Apache-2.0** —
  <https://github.com/stanford-crfm/helm/blob/main/LICENSE>
- License path in repo: `LICENSE`
- License SPDX: `Apache-2.0`
- Default branch: `main`
- Language: Python

## Install

```bash
pip install crfm-helm
# or with all extras (vision-language, medical, code, audio)
pip install "crfm-helm[all]"

# Run HELM Lite v1.0 against an OpenAI model
helm-run \
    --run-entries mmlu:subject=anatomy,model=openai/gpt-4o-mini \
    --suite my-lite-run --max-eval-instances 100

# Aggregate results and serve the HELM-style HTML leaderboard locally
helm-summarize --suite my-lite-run
helm-server --suite my-lite-run   # browse on :8000
```

## Niche

Multi-axis LLM evaluation — **not just accuracy, but accuracy +
calibration + robustness + fairness + bias + toxicity + efficiency
on the same run** — packaged as a leaderboard generator. Sits next
to [lm-evaluation-harness](../lm-evaluation-harness/),
[lighteval](../lighteval/), [inspect-ai](../inspect-ai/),
[deepeval](../deepeval/), [ragas](../ragas/), but its slot is *"I
want a HELM-shaped multi-metric leaderboard for my org's models"*,
not "give me a single MMLU number" or "score one RAG trace".

## Why it matters

- **Holistic = multiple metrics per scenario, not just accuracy** —
  every `RunSpec` reports `exact_match` *and*
  `expected_calibration_error`, *and* perturbation-robustness
  (typos, capitalisation, dialect rewrites), *and* fairness across
  demographic perturbations, *and* toxicity / bias when relevant.
  Single-metric leaderboards can't tell you the model that hits
  85% MMLU is also miscalibrated and brittle to typos; HELM does.
- **The leaderboard generator behind public HELM pages** — Stanford
  CRFM's own crfm-helm.stanford.edu / crfm-models.stanford.edu
  sites are produced by this same `helm-summarize` +
  `helm-server` toolchain. Running a private HELM-shaped
  leaderboard for an enterprise model fleet is a config change,
  not a re-implementation.
- **Domain-specialised suites under one harness** — MedHELM
  (clinical QA, MedCalc, MedDialog, EHR-shaped), HELM-VLM
  (vision-language: VQAv2, OK-VQA, VizWiz, MathVista),
  FinanceBench (10-K QA, financial reasoning), EnterpriseBench,
  HELM-Instruct (instruction-following), HELM-Safety
  (red-teaming + jailbreak resistance), HELM-CodeInsights, and a
  growing audio track. One install, one CLI, one leaderboard
  format across all of them.
- **Pick `helm` over [`lm-evaluation-harness`](../lm-evaluation-harness/)**
  when you want robustness / fairness / calibration metrics
  alongside accuracy on the same run, when domain suites
  (medical / financial / vision) are part of the eval, or when
  the deliverable is an HTML leaderboard not a JSON blob.
- **Pick `helm` over [`lighteval`](../lighteval/)** when
  multi-axis metrics + leaderboard generation matter more than
  multi-GPU throughput against HF checkpoints.
- **Pick `helm` over [`inspect-ai`](../inspect-ai/)** when the
  evaluation surface is *benchmark scoring with multi-axis
  metrics*, not agent-with-tool-use-in-Docker. The two compose:
  `inspect` for agentic / capability evals, `helm` for the
  multi-metric leaderboard view of the resulting model line.
- **Pick `helm` over [`deepeval`](../deepeval/) /
  [`ragas`](../ragas/)** when the system-under-test is a base
  model evaluated on academic + domain scenarios, not a RAG
  pipeline scored on faithfulness / context-precision.
- Apache-2.0, no telemetry; runs are JSON + the
  static-HTML leaderboard is rendered locally — no
  hosted-control-plane requirement, the trust profile matches
  the rest of the academic-eval cluster in this catalog.
- Caveat: `helm-run` is heavier than `lm_eval` or `lighteval` —
  scenarios pull large datasets, multi-metric runs evaluate the
  same prompt under multiple perturbations (so wall-time is a
  multiple of the single-metric path), and the HELM-Lite
  configuration is the one most users should start with rather
  than HELM-Classic.
