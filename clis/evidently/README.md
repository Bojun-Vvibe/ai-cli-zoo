# evidently

> Snapshot date: 2026-04. Upstream: <https://github.com/evidentlyai/evidently>

**Open-source ML + LLM observability framework with 100+ built-in
metrics.** Evidently runs as a Python library (`evidently`) plus an
optional self-hosted UI (`evidently ui`) that turns "evaluate / test /
monitor a data or AI pipeline" into a uniform `Report` + `TestSuite`
object you can render to HTML, JSON, or push to a project workspace.
For LLMs it ships descriptors (toxicity, sentiment, semantic
similarity, regex / LLM-as-judge) you compose into a `Dataset` +
`Report` and run on every batch.

## Repo + version + license

- Repo: <https://github.com/evidentlyai/evidently>
- Latest release: **`v0.7.21`** (2026-03-10)
- HEAD on `main`: `eb5d82a`
- License: **Apache-2.0** —
  <https://github.com/evidentlyai/evidently/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Python (Jupyter-heavy docs)

## Install

```bash
pip install -U evidently
# launch the local UI workspace (project + dashboards)
evidently ui --workspace ./workspace --host 0.0.0.0 --port 8000
# one-shot programmatic report
python -c "from evidently import Report; from evidently.presets import DataDriftPreset; \
  r = Report([DataDriftPreset()]); r.run(reference_data=ref, current_data=cur); r.save_html('report.html')"
```

## Niche

The "**one library for both classical-ML data drift and LLM judge
metrics**" slot. Where [ragas](../ragas/) and [deepeval](../deepeval/)
are LLM-eval-first and [mlflow](../mlflow/) is experiment-tracking-
first, Evidently sits on the *monitoring* side: a long-lived workspace
of projects, each holding dated snapshots, with the same `Report`
abstraction whether the input is a tabular feature matrix or a
conversation log. The `evidently ui` CLI gives you a self-hostable
dashboard without standing up Grafana / Prometheus.

## Why it matters

- 100+ built-in metrics across data quality, drift, classification,
  regression, ranking, recsys, and LLM evaluation — most production
  monitoring questions are answered without writing a metric class.
- LLM `Descriptor` system (`Sentiment`, `Toxicity`, `SemanticSimilarity`,
  `LLMEval` for judge prompts, `RegExp`) composes into the same
  `Dataset` + `Report` pipeline as tabular metrics, so a single
  workspace tracks a RAG service and its upstream embedding-feature
  drift side by side.
- `evidently ui` is a single-binary self-hosted dashboard (Apache-2.0,
  no SaaS lock-in) — pairs cleanly with cron / Airflow snapshots
  pushed via `Workspace.add_run()`.
- Outputs are also pure JSON / HTML, so the same report can be a CI
  artifact (fail the build on drift > X) and a dashboard tile.
