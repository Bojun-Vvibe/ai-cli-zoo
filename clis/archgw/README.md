# archgw (Plano)

- **Repo:** https://github.com/katanemo/archgw
- **Version:** 0.4.22 (latest release)
- **License:** Apache-2.0 (`LICENSE`)

## What it does

An AI-native proxy and data plane for agentic apps. Sits between your agent
code and upstream LLM providers, providing smart LLM routing, guardrails,
prompt-level observability, and orchestration primitives. Built on top of
Envoy with purpose-built small models for routing and safety.

## Install

```sh
brew install katanemo/archgw/archgw
# or via Docker
docker run -p 10000:10000 katanemolabs/archgw:latest
```

## Why it's interesting

Most LLM gateways are thin reverse proxies. archgw treats the gateway as
an application-aware data plane: it can rewrite prompts, enforce JSON
schemas, route between models per-intent, and emit OTel traces — all
declared in a single config file rather than scattered across agent code.
