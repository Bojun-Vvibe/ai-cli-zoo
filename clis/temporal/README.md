# temporal

> **Single-binary CLI for the Temporal durable-execution platform:**
> start a local dev server, register namespaces, drive workflows /
> activities, replay history, and operate a remote cluster — all
> from one Go binary. Pinned to **v1.7.0**
> ([LICENSE](https://github.com/temporalio/cli/blob/main/LICENSE),
> MIT).

Source: <https://github.com/temporalio/cli>

## TL;DR

`temporal` is the official command-line client for Temporal, a
workflow-orchestration engine where business logic is written as
ordinary code (Go / Java / Python / TypeScript / .NET / PHP / Ruby)
and the runtime persists every step's input, output, timer, and
side-effect to durable storage. The CLI bundles three things in
one ~80 MB binary: a **dev server** (`temporal server start-dev`)
that runs the full backend in-process against an embedded SQLite
DB so you can develop offline; an **operator** surface
(`temporal operator namespace`, `... cluster`, `... search-attribute`)
for cluster admin; and a **workflow client**
(`temporal workflow start|describe|list|signal|query|terminate`,
`temporal activity ...`, `temporal task-queue ...`) for driving
running workflows. Output is human-readable by default and
machine-readable with `-o json|jsonl|table`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install temporal

# install script (Linux / macOS)
curl -sSf https://temporal.download/cli.sh | sh

# from source (requires Go 1.22+)
go install github.com/temporalio/cli/cmd/temporal@v1.7.0

# verify
temporal --version    # temporal version 1.7.0
```

The CLI is statically linked, has no runtime dependencies, and
talks to a Temporal frontend over gRPC (default `127.0.0.1:7233`).
For local development the bundled dev server is a single command;
for remote work, set `TEMPORAL_ADDRESS` + `TEMPORAL_NAMESPACE` (or
pass `--address` / `--namespace`) and you are connected.

## License

MIT — see
[LICENSE](https://github.com/temporalio/cli/blob/main/LICENSE).
The CLI itself is MIT; the broader Temporal server
(`temporalio/temporal`) is also MIT. No CLA traps, no source-
available carve-outs for managed-service use.

## One Concrete Example

A 60-second loop: spin up a dev cluster, start a workflow, signal
it, query it, and inspect history.

```bash
# 1. start the in-process dev server (Web UI on :8233, gRPC on :7233)
temporal server start-dev --db-filename /tmp/temporal-dev.sqlite &
sleep 2

# 2. register a namespace + a custom search attribute
temporal operator namespace create demo --retention 24h
temporal operator search-attribute create \
    --namespace demo --name OrderStatus --type Keyword

# 3. start a workflow (assumes a worker is already polling task-queue "orders")
temporal workflow start \
    --namespace demo \
    --task-queue orders \
    --type ProcessOrder \
    --workflow-id order-2026-04-29-001 \
    --input '{"order_id":"A-001","items":3}'

# 4. send a signal mid-flight
temporal workflow signal \
    --namespace demo \
    --workflow-id order-2026-04-29-001 \
    --name AddNote \
    --input '"customer requested gift wrap"'

# 5. synchronous query against the running workflow's in-memory state
temporal workflow query \
    --namespace demo \
    --workflow-id order-2026-04-29-001 \
    --type GetCurrentStep

# 6. list every running workflow whose OrderStatus is "PENDING"
temporal workflow list \
    --namespace demo \
    --query 'OrderStatus = "PENDING"' \
    -o table

# 7. dump the full event history (for replay or post-mortem)
temporal workflow show \
    --namespace demo \
    --workflow-id order-2026-04-29-001 \
    --output json > order-A-001.history.json

# 8. replay that history against your current worker code
#    (verifies determinism after a refactor — fails loudly if you broke replay)
temporal workflow replay --from-file order-A-001.history.json
```

For automation, every command accepts `-o json` / `-o jsonl` and
returns non-zero on RPC failure, so it slots cleanly into
`Makefile`s, GitHub Actions, or an LLM agent's tool surface.

## Niche It Fills

**Operating a durable-execution workflow cluster from the
terminal, including a zero-config local cluster.** Most workflow /
job systems (Airflow, Prefect, Dagster) ship a web UI plus a thin
CLI that mostly wraps "trigger a DAG"; cluster admin requires the
UI or a Python API. Temporal's model is different — workflows are
arbitrary code, and the CLI is the *primary* interface for both
developers (local dev server, replay, signal, query) and operators
(namespace, search-attribute, cluster, task-queue). One binary
covers the whole loop, no Docker compose, no Postgres, no Helm.

## Why use it

1. **Local dev server in one command.** `temporal server start-dev`
   runs the frontend, history, matching, and worker services
   in-process against SQLite, plus the Web UI on `:8233`. Cold
   start is sub-second; teardown is `^C`. This collapses the
   "spin up Postgres + Cassandra + Elasticsearch" tutorial step
   into something you can put in a `make test` target.
2. **Replay-from-history is a first-class verb.**
   `temporal workflow replay --from-file <history.json>` runs your
   current worker code against a captured event history and fails
   if the new code makes a non-deterministic decision (different
   activity, different timer, different side-effect order). This
   is the canonical safety net before deploying a workflow refactor;
   no other orchestrator ships an equivalent CLI affordance.
3. **JSON output everywhere + scriptable visibility queries.**
   `--query` accepts a SQL-like predicate over search attributes
   (`WorkflowType = "ProcessOrder" AND OrderStatus = "PENDING" AND
   StartTime > "2026-04-01T00:00:00Z"`), and `-o jsonl` streams
   one record per line. An agent can `temporal workflow list
   --query ... -o jsonl | jq ...` to pick targets, then
   `temporal workflow signal` / `terminate` / `reset` against each.

## Vs Already Cataloged

- **Vs [`dagster`](../dagster/) / [`prefect`](../prefect/) /
  [`metaflow`](../metaflow/):** those are Python-first DAG
  schedulers — you define a graph of tasks, the scheduler runs
  them, and retries are at the task boundary. Temporal is
  *durable-execution*: your workflow is straight-line code (loops,
  conditionals, `await`), and the runtime persists each step so the
  whole function survives crashes, deploys, and week-long sleeps.
  Pick a DAG scheduler for ETL / batch / data-pipeline shapes; pick
  Temporal for long-lived business processes (orders, onboarding,
  approvals, sagas).
- **Vs [`argocd`](../argocd/) / [`argo-rollouts`](../argo-rollouts/):**
  those are Kubernetes-native delivery / progressive-rollout tools
  bound to k8s objects. Temporal is platform-agnostic — workers
  run anywhere they can dial out to the frontend gRPC port, and
  the CLI has no k8s assumptions.
- **Vs [`flux`](../flux/):** Flux is GitOps reconciliation for
  k8s manifests. Temporal is application-level workflow
  orchestration. They compose (Flux deploys your worker pods,
  Temporal schedules the work the pods do), they do not overlap.
- **Vs [`nomad`](../nomad/):** Nomad is a workload scheduler
  (place a container / batch job on a host). Temporal is a
  workflow engine on top of whatever already places your worker
  process. Different layer.

## Caveats

- **You still need a worker process.** The CLI starts workflows
  but does not execute their code; an SDK-built worker
  (Go / Java / Python / TS / .NET / PHP / Ruby) must be polling
  the task queue or your `workflow start` invocation will queue
  forever. This is by design — workflow logic is your code, not
  the CLI's — but new users sometimes expect `temporal workflow
  start` to also run the workflow.
- **`server start-dev` is not for production.** SQLite persistence,
  no horizontal scale, no auth, no TLS. For production you run
  the full `temporalio/temporal` server against Postgres / MySQL /
  Cassandra with Elasticsearch (or its embedded equivalent) for
  visibility, and the CLI just connects to it.
- **gRPC means strict version skew rules.** A 1.7.x CLI talks to
  any 1.x server, but a workflow worker built against an old SDK
  may emit history events the CLI cannot pretty-print on
  `workflow show`; fall back to `--output json` and parse with
  `jq`. Operator commands (namespace, search-attribute) are the
  most version-sensitive — check the CLI's `--help` for the
  flag set on your server's version.
- **Search attributes need the visibility store.** `--query` only
  works against fields registered as search attributes
  (`temporal operator search-attribute create`) and only after
  the visibility store (Elasticsearch in production, SQLite's
  built-in visibility in dev) has indexed the workflow. Newly
  started workflows may take a second or two to become
  list-queryable.
