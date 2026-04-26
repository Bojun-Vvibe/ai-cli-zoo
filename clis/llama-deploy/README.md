# llama-deploy

- **Repo:** https://github.com/run-llama/llama_deploy
- **Version:** `v0.9.2` (latest release)
- **License:** MIT (`LICENSE`)
- **Language:** Python
- **Install:** `pip install llama-deploy` then `llamactl serve` /
  `llamactl deploy ./my-workflow`

## One-line summary

A deployment runtime for **LlamaIndex Workflows as long-running services** —
turn a Python `Workflow` class (event-driven, async, with steps) into an
HTTP API + control plane with a message queue, multi-replica scaling, and
streaming results, without rewriting it as FastAPI.

## What it does

LlamaIndex Workflows are an event-driven agent abstraction (steps wait on
events, emit events, can branch / loop / fan-out). `llama-deploy` wraps that
abstraction in production scaffolding:

- **Apiserver**: HTTP gateway that accepts workflow runs, streams events
  back via SSE, and exposes per-deployment OpenAPI specs.
- **Message queues**: pluggable transport between the apiserver and
  workflow workers — in-memory for dev, Redis / Kafka / RabbitMQ /
  AWS SQS / Apache Kafka for prod.
- **`llamactl` CLI**: `llamactl serve` boots the local apiserver,
  `llamactl deploy ./workflow.yaml` registers a workflow as a named
  deployment, `llamactl run my-deployment --arg "..."` invokes it with
  streaming output.
- **Service composition**: a workflow step can call another deployed
  workflow over the message bus (`ServiceComponent`), so a "router"
  workflow can fan out to specialist workflows on different replicas /
  models.

The deployment unit is a directory with a `pyproject.toml` + a workflow
YAML pointing at the `Workflow` subclass — `llamactl deploy` packages and
registers it without rebuilding a container.

```bash
pip install llama-deploy
llamactl serve &                          # apiserver on :4501
llamactl deploy ./my-workflow            # register
llamactl run my-workflow -i '{"q":"hi"}' # stream events
```

```yaml
# my-workflow/llama_deploy.yaml
name: my-workflow
default-service: my_service
services:
  my_service:
    name: My Service
    source:
      type: local
      name: .
    path: workflow:my_workflow
```

## When to choose it

- You're already building agents as **LlamaIndex Workflows** and want a
  thin path from notebook → HTTP service without rewriting as FastAPI +
  Celery.
- You need **multi-workflow composition over a message bus** — one
  router workflow dispatching to specialist workflows on independent
  scaling groups.
- You want **streaming events to the client** (SSE) as the default
  shape, not request/response with polling.

## When NOT to choose it

- Your agent runtime is anything other than LlamaIndex Workflows —
  llama-deploy is tightly coupled to that abstraction.
- You want a fully managed serverless deployment — this is
  self-host-shaped (apiserver + queue + workers); use the LlamaCloud
  product for managed hosting.
- You need durable workflow execution with replay across process crashes
  comparable to `inngest` / Temporal — llama-deploy's durability story
  depends on the message queue, not first-class checkpointed steps.
