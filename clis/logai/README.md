# logai

> Snapshot date: 2026-04. Upstream: <https://github.com/salesforce/logai>
> Binary name: `logai` (CLI) and `logai-gui` (Streamlit UI)

`logai` is the **log-analysis CLI** of the catalog. It is the only
entry whose primary input is application / system log files — gigabytes
of unstructured text — and whose primary output is structured
findings: parsed log templates, anomaly clusters, ranked
"interesting" lines, and (optionally) LLM-authored explanations of
what each anomaly cluster means.

The pipeline is intentionally not "shove the whole log file at GPT".
Instead `logai` runs a classical log-mining stack first — Drain /
IPLoM template extraction, TF-IDF / log2vec embeddings, Isolation
Forest / DBSCAN anomaly scoring — and *then* hands the top-N
anomalous templates to an LLM for natural-language summarization.
This is the AI-native angle: the model sees ~50 representative
lines, not 50 million, so context cost stays bounded and the
explanations are grounded in real templates rather than
hallucinated.

```
logai workflow --config config.yaml          # run the full pipeline
logai workflow --task anomaly_detection \
  --log-file /var/log/app.log                # one-shot anomaly scan
```

A `config.yaml` declares the parser (`Drain`, `IPLoM`, `AEL`),
vectorizer (`TfIdf`, `Word2Vec`, `Semantic`), and detector
(`IsolationForest`, `LOF`, `DBSCAN`), so you can swap algorithms
without touching code.

## 1. Install footprint

- Python 3.8+. `pip install logai` pulls scikit-learn, gensim,
  pandas, and (optionally) torch + transformers for the semantic
  embedding path. Heavy install (~1.5 GB with all extras), light
  install (~150 MB) without semantic models.
- Optional `logai[deep-learning]` extra installs PyTorch for the
  semantic vectorizer.
- Bring-your-own API key for the LLM summarization step
  (`OPENAI_API_KEY`); pure-classical workflows need no key at all.

## 2. License

BSD-3-Clause.

## 3. Models supported

- **Embedding/anomaly:** TF-IDF, Word2Vec, FastText, sentence-
  transformers, all running locally.
- **Summarization (optional):** OpenAI, any OpenAI-compatible
  endpoint via `OPENAI_API_BASE` (Ollama, vLLM, LiteLLM).

## 4. MCP support

None. Pipeline tool, not an agent.

## 5. Sub-agent model

None. The pipeline stages run sequentially; only the final
summarization stage talks to an LLM.

## 6. Telemetry stance

Off. No analytics. Egress only if you enable the LLM summarization
stage.

## 7. Prompt-cache strategy

Templates are deduplicated before being shown to the model — the
implicit "cache" is the log-template extraction itself, which
collapses millions of lines to dozens of templates.

## 8. Hot keybinds

CLI is non-interactive. The companion `logai-gui` is a Streamlit
app with point-and-click pipeline configuration.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Two-stage architecture — classical log mining
collapses volume, LLM only sees the residual anomalies. This is the
right shape for incident response: you can run it on a 10 GB log
file in minutes for under a dollar of LLM spend, where a
"summarize this file" approach would either blow the context window
or cost hundreds of dollars.

**Weakness.** The classical-mining defaults assume you have *enough*
log volume for templates and clusters to mean something — on a few
hundred lines of output it has nothing to say. The Streamlit UI is
the more polished surface; the bare CLI is config-file-driven and
has a learning curve.

**When to choose.** Production incident, you have a haystack of
logs and need to find the needle, and you want the explanation in
plain English. Pair with [`fabric`](../fabric/) if you want to
post-process the JSON output through a custom prompt pattern, or
[`llm`](../llm/) if you want to log every summarization call to
SQLite for the post-mortem.

**Niche vs neighbors.** No other entry in the catalog touches log
analysis. [`marker`](../marker/) converts PDFs, [`repomix`](../repomix/)
packs source code, [`files-to-prompt`](../files-to-prompt/) packs
text trees — none of them do template extraction or anomaly
scoring on streaming log volume. `logai` is the only "operational
data → AI" entry in the zoo.
