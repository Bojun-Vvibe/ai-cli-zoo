# steampipe

> **Query cloud APIs, SaaS, and config files with SQL** —
> one Postgres-compatible engine that loads "plugins" (AWS,
> GCP, Azure, GitHub, Kubernetes, Slack, CSV, JSON, …) as
> foreign tables and lets you `SELECT name, region, instance_type
> FROM aws_ec2_instance WHERE state = 'running'` against your
> live account, with results joinable across providers in a
> single query. Pinned to **v1.1.3**
> ([LICENSE](https://github.com/turbot/steampipe/blob/main/LICENSE),
> AGPL-3.0).

Source: <https://github.com/turbot/steampipe>

## TL;DR

`steampipe` boots an embedded PostgreSQL (FDW-based) where every
plugin you install registers a schema of read-only tables backed
by the underlying API. `steampipe plugin install aws` adds
`aws_ec2_instance`, `aws_s3_bucket`, `aws_iam_user`, … (~400
tables); `steampipe plugin install github` adds `github_my_repository`,
`github_pull_request`, `github_workflow_run`, etc. Then
`steampipe query` drops you into a psql-like REPL where you write
ordinary SQL — joins, aggregations, CTEs, window functions —
across providers (`SELECT ... FROM aws_ec2_instance i JOIN
github_repository r ON i.tags->>'Repo' = r.name`). Results are
cached on disk (default 5 min TTL) so re-running the same query
doesn't re-hit the API. The engine speaks the Postgres wire
protocol on `localhost:9193`, so `psql`, `pgcli`, BI tools,
Grafana, and Metabase all work without modification.

## Install

```bash
# Homebrew
brew install turbot/tap/steampipe

# install script (Linux/macOS)
sudo /bin/sh -c "$(curl -fsSL https://steampipe.io/install/steampipe.sh)"

# verify
steampipe -v   # v1.1.3

# add a plugin (auth via env or ~/.aws/credentials picked up automatically)
steampipe plugin install aws
steampipe plugin install github
```

## One Concrete Example

```bash
# Inventory: every public S3 bucket with no default encryption,
# joined to the GitHub repo whose tag claims to own it.
steampipe query <<'SQL'
WITH unencrypted AS (
  SELECT
    name        AS bucket,
    region,
    tags->>'Repo' AS repo_slug
  FROM aws_s3_bucket
  WHERE server_side_encryption_configuration IS NULL
    AND (block_public_acls IS DISTINCT FROM true
      OR block_public_policy IS DISTINCT FROM true)
)
SELECT
  u.bucket,
  u.region,
  u.repo_slug,
  r.html_url           AS repo_url,
  r.pushed_at          AS last_pushed,
  r.archived           AS repo_archived
FROM unencrypted u
LEFT JOIN github_repository r
  ON r.full_name = u.repo_slug
ORDER BY r.pushed_at NULLS FIRST;
SQL

# Same query, machine-readable, for a CI gate
steampipe query --output csv  -f bucket-audit.sql > findings.csv
steampipe query --output json -f bucket-audit.sql | jq '.[] | select(.repo_archived)'
```

The trick is the *cross-provider join*: there's no other tool
where one SQL statement spans an AWS account and a GitHub org and
a Kubernetes cluster. Each provider on its own has a CLI that
returns JSON; `jq` + bash + careful `--query` flags can fake this,
but past three sources you're writing a small program. Steampipe
makes it a SELECT.

## Niche It Fills

**SQL as a universal query language for read-only cloud/SaaS
inventory.** The job-to-be-done is "answer ad-hoc questions about
my infrastructure that span 2-5 systems, today, without writing
code." Auditors, security engineers, FinOps, and SREs reach for
it when the question is shaped like "show me X where Y, joined
to Z" and the data already exists behind APIs they have
credentials for.

## Vs Already Cataloged

- **Vs [`osquery`](../osquery/):** osquery is SQL over *host*
  state (processes, sockets, kexts, file hashes); steampipe is
  SQL over *cloud/SaaS* state. Same query language, opposite
  data plane. Run osquery on the box, steampipe against the
  account.
- **Vs [`duckdb`](../duckdb/) + provider extensions:** DuckDB
  can read S3, JSON, Parquet, and via community extensions some
  cloud catalogs, but it has no plugin model that maps live API
  calls to SQL tables — you'd be hand-writing `httpfs` queries
  per endpoint. Use DuckDB for analytical queries over data you
  already extracted; use Steampipe to do the extracting and
  joining live.
- **Vs `aws ec2 describe-instances --query` /
  [`gh`](../gh/) / `kubectl`:** the per-provider CLIs are great
  for one provider at a time. Steampipe is what you reach for the
  moment a question needs two providers in one answer (or three,
  or five).
- **Vs [`cloudquery`](https://www.cloudquery.io/):** the closest
  competitor and the obvious alternative. CloudQuery is an
  *ETL* — it writes provider data into Postgres/BigQuery/ClickHouse
  on a schedule and you query the snapshot. Steampipe is *live*
  — every SELECT hits the API (or the 5-min cache). Pick
  CloudQuery for analytical workloads, dashboards, and historical
  trends; pick Steampipe for "what's true right now."
- **Vs [`k8sgpt`](../k8sgpt/) / cloud-native SaaS scanners:**
  scanners ship opinionated rules and verdicts. Steampipe ships
  a query engine; you (or `powerpipe`, the companion compliance
  tool from the same vendor) write the rules. More flexible,
  less out-of-the-box.

## Caveats

- AGPL-3.0 — vendoring the binary into a closed-source product
  triggers the network-use clause. For internal CLI use and CI
  jobs this is irrelevant, but vendor/legal review is warranted
  before embedding.
- Plugins authenticate using each provider's *normal* credential
  chain (env vars, `~/.aws/credentials`, `gcloud` ADC, GitHub
  PATs). That means whatever scope your shell can reach,
  steampipe can too — run it under the lowest-privilege role you
  can, especially in shared dev environments.
- The 5-minute query cache is helpful for REPL exploration and
  pricey for "is this resource still there?" questions; pass
  `--cache=false` (or set `STEAMPIPE_CACHE=false`) when you
  need fresh reads.
- Some plugins paginate aggressively for large accounts; a
  naive `SELECT * FROM aws_cloudwatch_log_event` will try to
  fetch every event in every group. Always add `WHERE region =
  ...` and time-window predicates first, then widen.
- The embedded Postgres listens on `127.0.0.1:9193` by default.
  If port 9193 is already taken (rare), `steampipe service
  start --database-port 9293`. The service stays up between
  invocations — `steampipe service stop` to free resources.
- `powerpipe` (compliance dashboards) and `flowpipe` (event-driven
  workflows) are sibling tools from the same vendor that consume
  the same plugin set; they're separate binaries and out of
  scope here, but worth knowing if you outgrow ad-hoc queries.
