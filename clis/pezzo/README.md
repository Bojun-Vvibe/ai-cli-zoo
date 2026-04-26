# pezzo

> Snapshot date: 2026-04. Upstream: <https://github.com/pezzolabs/pezzo>

**Open-source, developer-first LLMOps platform centred on prompt management,
versioning, observability, and instant prompt delivery.** Pezzo is a self-
hostable Nx monorepo (Node + NestJS server, Next.js console, TypeScript +
Python client SDKs) that treats prompts as deployable artifacts: edit in
the console, publish to an environment label (`Production` / `Staging` /
`Development`), and your running services pick up the new version on the
next request without a redeploy. Every LLM call routed through the SDK
becomes a request log with prompt-version + cost + latency + token counts.

## Repo + version + license

- Repo: <https://github.com/pezzolabs/pezzo>
- Latest release: **`v0.9.2`** (2024-05-15)
- HEAD on `main`: `886d38b`
- License: **Apache-2.0** —
  <https://github.com/pezzolabs/pezzo/blob/main/LICENSE>
- License path in repo: `LICENSE` (11356 bytes, full Apache-2.0 text)
- Default branch: `main`
- Language: TypeScript (NestJS server, Next.js console, TS + Python SDKs)

## Install

```bash
# Self-host via the bundled docker compose stack
git clone https://github.com/pezzolabs/pezzo && cd pezzo
docker compose -f docker-compose.infra.yaml up -d   # Postgres + Redis + Clickhouse + Supertokens
pnpm install && pnpm migrate && pnpm dev            # console at :3000, server at :3001
```

```ts
import { Pezzo, PezzoOpenAI } from "@pezzo/client";
const pezzo = new Pezzo({ apiKey: process.env.PEZZO_API_KEY!, projectId: "...", environment: "Production" });
const openai = new PezzoOpenAI(pezzo, { apiKey: process.env.OPENAI_API_KEY! });
const prompt = await pezzo.getPrompt("ProductDescriptionGenerator");
const r = await openai.chat.completions.create(prompt);   // call is recorded with prompt version
```

## Niche

The "**self-hostable prompt CMS with environment-label deploys + a thin
observability layer**" slot. Where [`langfuse`](../langfuse/) bundles
prompt management into a much larger product (traces + evals + datasets +
playground + a heavy Postgres / Clickhouse / Redis / MinIO operational
footprint) and where [`helicone`](../helicone/) leads with one-line
base-URL-swap polyglot tracing, Pezzo's bet is the *prompt-as-deployable*
workflow: prompt engineers ship a new system prompt by clicking
"Publish to Production" in the console, and the next call site invocation
picks it up. The trade-off is reach — only the OpenAI provider has a
first-class wrapped client (`PezzoOpenAI`); other providers are
manually-instrumented or left to a third-party tracer.

## Why it matters

- **Prompts have versions, environments, and a publish step** — `prompt
  v12` lives in `Development`, gets promoted to `Staging`, then
  `Production`; the SDK call `pezzo.getPrompt("Foo")` always returns
  the version published to the SDK's configured environment, so a
  non-engineer prompt edit ships without a code deploy and rolls back
  in one click.
- **Settings travel with the prompt** — model, temperature, top-p,
  max_tokens, response_format land in the same artifact as the prompt
  text, so prompt + decoding params version together (the common
  failure mode where someone bumps `temperature` in code and forgets
  to roll back doesn't apply).
- **Bundled observability for the calls it sees** — every request
  through `PezzoOpenAI` lands in Clickhouse with prompt version, model,
  cost (computed from a maintained price table), latency, and token
  counts; filterable in the console for debugging "which prompt
  version regressed cost overnight".
- **Apache-2.0 across the whole monorepo** — no EE carve-out;
  self-hosting the entire console + server + observability stack is
  legally simple. Caveat — project velocity has slowed (latest tag
  v0.9.2 dates to 2024-05); treat as a stable mature LLMOps stack
  rather than a fast-moving one, and pin the image SHA in production.
