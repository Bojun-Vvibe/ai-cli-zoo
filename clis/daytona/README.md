# daytona

> Snapshot date: 2026-04-26. Upstream: <https://github.com/daytonaio/daytona>

"**Secure and Elastic Infrastructure for Running AI-Generated
Code.**" Daytona is a sandbox runtime purpose-built for agent
workloads: spin up isolated Linux VMs in milliseconds, give your
agent a shell + filesystem + network policy, and tear it down
when the task is done. Originally a dev-environment manager,
Daytona pivoted to **AI sandbox** as the primary use case in
2025 and now ships SDKs that are drop-in replacements for the
hosted sandbox APIs agent frameworks default to.

## 1. Install footprint

- CLI: `curl -fsSL https://download.daytona.io/daytona/install.sh | bash`
  installs the `daytona` binary; `daytona serve` starts the local
  server, `daytona sandbox create` provisions a sandbox.
- SDKs: `pip install daytona-sdk` (Python) and
  `npm install @daytonaio/sdk` (TypeScript) expose
  `Sandbox.process.exec`, `Sandbox.fs.*`, `Sandbox.git.*` so an
  agent can shell out, read/write files, and clone repos against
  the sandbox.
- Self-host: full stack runs via `docker compose` or Helm chart;
  hosted tier at `app.daytona.io` for teams that don't want to
  operate the runtime.

## 2. Repo + version + license

- Repo: <https://github.com/daytonaio/daytona>
- Latest release: **v0.169.0** (2026-04-23)
- License: **AGPL-3.0** —
  <https://github.com/daytonaio/daytona/blob/main/LICENSE>
- Default branch: `main`
- Language: TypeScript / Go
- Stars: ~72k

## 3. Models supported

Daytona itself is **model-agnostic infrastructure** — it runs
sandboxes, not models. The matrix it cares about is the agent
framework on top: the SDK is integrated with LangChain, LlamaIndex,
CrewAI, AutoGen, OpenAI Agents SDK, and Anthropic's tool-use loop,
so any model those frameworks support can drive a Daytona
sandbox as its execution surface.

## 4. Notable angle

**Sub-second cold-start + AGPL self-host as the differentiator
versus E2B / Modal sandbox tiers.** Daytona's pitch is that you
get the same "give the agent a real Linux box" UX as the hosted
incumbents but you can run the control plane yourself, keep code
on your own hardware, and avoid the per-second billing surprise
when a runaway agent loops. Snapshots + image caching push cold
start under a second for most workflows, which makes it usable
as a per-tool-call sandbox, not just a per-session one. AGPL
licensing is deliberate: SaaS forks must publish their changes,
single-tenant self-host is unrestricted.

## 5. Last verified

2026-04-26 via `gh api repos/daytonaio/daytona`.
