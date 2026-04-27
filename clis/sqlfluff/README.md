# sqlfluff

> The dialect-aware **SQL linter and auto-formatter** for the modern data
> stack: parses SQL into a real syntax tree per dialect (Postgres, BigQuery,
> Snowflake, Redshift, Databricks, DuckDB, Trino, Athena, Spark, T-SQL,
> Teradata, MySQL, SQLite, Hive, Exasol, Materialize, ClickHouse, …),
> applies a configurable rule set, and rewrites the file in place. Pinned
> to **v4.1.0** (commit `939af547ebc238fcac9681cad137992add1ac703`,
> [LICENSE.md](https://github.com/sqlfluff/sqlfluff/blob/main/LICENSE.md), MIT).

Source: <https://github.com/sqlfluff/sqlfluff>

## TL;DR

`sqlfluff` is the `prettier` / `ruff` of SQL. `sqlfluff lint
queries/` walks a tree of `.sql` files and reports rule violations
(`L010` keyword case, `L030` function-name case, `L042` subquery
inside JOIN, `L036` SELECT-target newline, …) one finding per line
in a parseable format suitable for CI. `sqlfluff fix queries/`
rewrites those files in place using the same rule set so the diff
is reviewable in a PR. `sqlfluff format` is the no-questions-asked
formatter (subset of fix). `sqlfluff parse query.sql` dumps the
parse tree, which is the escape hatch when you need to know exactly
why a rule fired. The dialect is set per-project in
`.sqlfluff` (`dialect = bigquery`) and **templater** plugins
(`jinja`, `dbt`, `python`, `placeholder`) render the file before
parsing — so it lints `dbt` models with `{{ ref('foo') }}` and
parameterized queries with `:user_id` correctly, not as a syntax
error.

## Install

```bash
# pip
pip install -U sqlfluff              # 4.1.0

# uv
uv tool install sqlfluff

# verify
sqlfluff --version
```

Python 3.9+. Optional extras: `pip install 'sqlfluff[dbt]'` for the
`dbt` templater (uses your local `dbt-core` to render `{{ ref }}` /
`{{ source }}` / macros).

## One Concrete Example

`.sqlfluff` (project root):

```ini
[sqlfluff]
dialect = postgres
templater = jinja
exclude_rules = L034

[sqlfluff:rules:capitalisation.keywords]
capitalisation_policy = upper
```

`models/users.sql`:

```sql
select id, name, created_at
from users where deleted_at is null
order by id
```

Run:

```bash
sqlfluff lint models/users.sql
# L:   1 | P:   1 | LT01 | Expected single space before 'id'.
# L:   1 | P:   1 | CP01 | Keywords must be consistently upper case.

sqlfluff fix models/users.sql --force
# rewrites to:
# SELECT
#     id,
#     name,
#     created_at
# FROM users
# WHERE deleted_at IS NULL
# ORDER BY id
```

CI integration:

```bash
sqlfluff lint --format github-annotation \
  --annotation-level failure models/ > annotations.json
```

## Niche It Fills

**The deterministic SQL gatekeeper for an analytics monorepo.** Coding
agents in this catalog will happily write SQL, and `dbt`-style
projects accumulate hundreds of `.sql` files — without a linter the
style drifts, ambiguous joins land in production, and PR review
turns into bikeshedding over UPPERCASE vs lowercase keywords.
`sqlfluff` makes the answer mechanical: rules in `.sqlfluff`, fix
in pre-commit, lint in CI. It is the only entry in this catalog
whose job is *grading* SQL — paired with [`vanna`](../vanna/) /
[`wrenai`](../wrenai/) (which generate SQL from natural language)
or [`harlequin`](../harlequin/) (which lets a human edit it), it
catches the dialect-specific footguns the LLM does not know about.

## Vs Already Cataloged

- **Vs [`harlequin`](../harlequin/):** `harlequin` is an interactive
  TUI for *running* SQL against eight engines; `sqlfluff` is a
  batch linter / formatter that never connects to a database.
  They compose: lint in pre-commit, run in `harlequin`.
- **Vs [`sqlite-utils`](../sqlite-utils/):** `sqlite-utils` is for
  *manipulating* SQLite (insert, transform, query); `sqlfluff` is
  for *checking* SQL text against a style guide. Different axes.
- **Vs an LLM-driven reviewer ([`code-review-gpt`](../code-review-gpt/)):**
  the LLM reviewer is good at semantic SQL bugs ("this join will
  fan out"); `sqlfluff` is good at the deterministic, dialect-aware
  surface ("this is not valid BigQuery", "keyword case violates
  policy"). Run both — they catch different defects.

## Caveats

- Rules are **opinionated**; expect to disable 5–10 of the default
  set in `.sqlfluff` to match your house style. The `core` rule
  set is the safest starting point (`rules = core` in config).
- The Jinja / dbt templaters render the file before parsing — if
  rendering fails (missing macro, undefined var), the lint fails
  with a templating error, not a SQL error. Run `sqlfluff render
  path/to/model.sql` to debug.
- `sqlfluff fix` rewrites files in place. Always run inside a clean
  git working tree, or use `--check` first to preview.
- The dialect parser is a real LL parser per dialect, not regex —
  which means it is slower than `flake8`-class linters on large
  trees (~1s per 100 medium files). Cache via `pre-commit` hooks
  rather than re-linting the whole tree on every save.
