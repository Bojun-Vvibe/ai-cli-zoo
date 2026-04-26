# pandas-ai

> Snapshot date: 2026-04. Upstream: <https://github.com/sinaptik-ai/pandas-ai>
> Version pinned at write-time: **v3.0.0**
> License: **MIT for the core**, custom commercial license for `pandasai/ee/`
> (file: [`LICENSE`](https://github.com/sinaptik-ai/pandas-ai/blob/main/LICENSE),
> SPDX classified as `NOASSERTION` because of the dual-license layout)

`pandas-ai` (a.k.a. PandasAI) is a Python library + CLI from Sinaptik
that lets you **chat with a dataframe, a CSV, a parquet file, or a SQL
database** using a natural-language prompt that gets compiled to
pandas / SQL under the hood. The 3.x line reorganised the project
around a "smart datalake" abstraction with RAG over schemas and
multi-table joins.

It is the right tool when you have **tabular data and want
natural-language analytics** without writing the pandas yourself, and
you are willing to let an LLM author and execute Python over your data.

## 1. Install

```bash
pip install pandasai

# Optional: extras for specific connectors
pip install "pandasai[postgres]"
pip install "pandasai[snowflake]"
```

Single Python package. Configuration lives in
`~/.pandasai/config.json` plus environment variables for the model
provider (`PANDASAI_API_KEY`, `OPENAI_API_KEY`, etc.).

## 2. Simple invocation

```python
import pandas as pd
from pandasai import SmartDataframe

df = pd.read_csv("sales.csv")
sdf = SmartDataframe(df)

# Natural-language query over the dataframe
print(sdf.chat("Which 5 products had the highest revenue last quarter?"))
```

```bash
# CLI helper for ad-hoc queries against a CSV
pai --file sales.csv --query "top 5 products by revenue last quarter"
```

## 3. Why it lives in this catalog

- It occupies a niche the coding-agent CLIs do not: **the agent's
  output is an answer about your data, not a code edit**. The model
  writes pandas/SQL, runs it in a sandboxed Python process, and
  returns the result.
- The 3.x SmartDatalake layer is one of the few open-source examples
  of a usable text-to-SQL system that handles multi-table joins
  without requiring you to hand-write a schema doc.
- The license split (MIT core, commercial `ee/`) is worth knowing
  before adopting it: anything inside `pandasai/ee/` is **not
  redistributable** under MIT.

## 4. What it is not

Not a coding agent and not a general chat CLI. It will not edit
your repo, will not browse, will not autonomously install
dependencies. Compare against `vanna` (text-to-SQL only), `dataline`,
`chartdb`, and the various notebook-agent tools — `pandas-ai` is the
oldest and most batteries-included of this cohort.
