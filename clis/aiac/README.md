# aiac

> Snapshot date: 2026-04. Upstream: <https://github.com/gofireflyio/aiac>

A single-binary Go CLI that generates **infrastructure-as-code**
artifacts — Terraform, Pulumi, CloudFormation, Kubernetes manifests,
Dockerfiles, GitHub Actions workflows, Ansible playbooks, shell
scripts — from a one-line natural-language description, and writes
the result to a file (or stdout) ready to `terraform plan` or
`kubectl apply`.

Aiac fills a niche none of the existing entries in this catalog
target directly: it is **scoped to declarative infra** rather than
to application source code. The prompts, the post-processing (it
strips markdown fences and explanatory prose so the output is
valid HCL/YAML/Dockerfile-syntax on first try), and the convention
choices (provider blocks, variable conventions, sane defaults for
the target IaC tool) are all infra-shaped.

## 1. Install footprint

- `brew install aiac`, `go install github.com/gofireflyio/aiac/v5@latest`,
  Docker image (`ghcr.io/gofireflyio/aiac`), or a release binary.
- Single static Go binary, ~15 MB. No runtime deps.
- Config at `~/.config/aiac/aiac.toml` (TOML, multi-backend). You
  can also drive everything from CLI flags / env vars without a
  config file.
- No daemon. Each invocation is one LLM call (plus optional
  follow-up turns in chat mode).

## 2. License

Apache-2.0.

## 3. Models supported

Multi-backend, configured per-named-backend in TOML:

- OpenAI (`type = "openai"`)
- Azure OpenAI (`type = "openai"` with `url = "https://...azure..."`)
- Anthropic (`type = "anthropic"`)
- Amazon Bedrock (`type = "bedrock"`, IAM-auth)
- Ollama (`type = "ollama"`)
- Any OpenAI-compatible endpoint (LiteLLM, vLLM, Groq, Together,
  OpenRouter, DeepSeek) via `type = "openai"` + custom `url`.

You name backends in config (`default_backend = "claude"`), pick at
the CLI with `-b/--backend <name>`, and pick a model within the
backend with `-m/--model <name>`. Verify the backend type strings
on your installed version before relying — the v4→v5 migration
renamed several.

## 4. MCP support

None. Aiac does not consume tools and does not call out to other
processes mid-generation. The output is a text artifact you take
to `terraform` / `kubectl` yourself.

## 5. Sub-agent model

None. There is a `chat` subcommand for follow-up turns ("now add
encryption at rest", "now make the bucket versioned") that maintain
in-process history, but it is single-thread, single-context. No
planner/builder split, no parallel sub-agents, no tools.

## 6. Telemetry stance

**Off, no analytics SDK.** Outbound calls are exclusively to the
configured backend (the LLM provider). The Firefly-hosted
"governance" features in some versions are opt-in and require
explicit account configuration — verify on your installed version
before relying.

## 7. Prompt-cache strategy

None at the aiac layer. Single-shot prompts are small (one user
sentence + a system prompt that primes the IaC convention), so
caching would not help materially. Provider-side prefix caching
applies for free if you target Anthropic / Gemini and reuse the
same backend across many invocations in a chat session, but aiac
does not control or expose it.

## 8. Hot keybinds / commands

Not a TUI. Two main shapes:

- **One-shot generation:**
  - `aiac get terraform "an EKS cluster with 3 t3.medium nodes in
    us-east-1, private subnets, IRSA enabled"`
  - `aiac get dockerfile "a multi-stage build for a Rust service,
    distroless final image"`
  - `aiac get k8s "a Deployment + Service for an nginx with 3
    replicas, resource limits, and a PodDisruptionBudget"`
  - `aiac get github_action "lint Go code on PR with golangci-lint
    and cache modules"`
  - `aiac get cicd_jenkinsfile "..."` etc.
- **Chat mode** for iterative refinement:
  - `aiac chat -t terraform` — opens a stateful REPL where each
    turn is added context. `.exit` to leave.
  - `.save <path>` (or `-o <path>` on the spawning command) writes
    the latest assistant turn to disk.

Useful flags on `get`:

- `-o <path>` write to file instead of stdout
- `-q` / `--quiet` strip the "Generated with aiac" preamble
- `-s` / `--save-id <id>` saves the run to local history
  (verify before relying)
- `-b <backend> -m <model>` route this call to a specific
  backend/model

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Output is a clean infra artifact, not a
chat transcript.** General-purpose LLMs answering "write me a
Terraform module for an EKS cluster" produce a markdown blob with
explanation paragraphs, code fences, and "you'll also want to
configure …" tail-text. You then hand-extract the HCL, fix the
fencing, drop the prose. Aiac's system prompts and post-processing
make the model emit *only* the artifact, in syntactically-valid
form, ready to `terraform fmt && terraform validate` immediately.
That post-processing layer is the entire product, and it
generalizes across ten artifact types.

The chat-mode iterative refinement loop ("now add tags", "now
parametrise the region", "now switch the bucket to S3-managed
encryption") is the second reason to use it: each turn rewrites
the file in place, you `git diff` between turns to see what
changed.

**Weakness.** **No state, no plan, no apply.** Aiac generates the
artifact and stops. It will not run `terraform plan` against your
state file, will not detect drift, will not refactor an existing
module to match a new spec, and will not learn your repo's
conventions (variable naming, module layout, provider versions)
from the surrounding files. So for **greenfield modules** it is
excellent; for **editing an existing 5-file Terraform module** you
are better off with `aider` or `claude-code` pointed at the
directory, because those see the existing code and aiac does not.

Quality is also **provider-shape sensitive**: it knows the common
shape of an EKS cluster, an RDS instance, an S3 bucket. It is much
weaker on niche resources (third-party Terraform providers, custom
Pulumi components, in-house Helm charts) where the model has thin
training data and aiac has no repo context to compensate.

**When to choose.**

- You need to **scaffold a new IaC artifact from scratch** — a new
  Terraform module, a new K8s manifest set, a new Dockerfile, a
  new GitHub Actions workflow — and you want syntactically clean
  output you can `fmt && validate` without surgery.
- You want a **single binary, no Python venv**, that drops into a
  shell pipeline (`aiac get dockerfile "…" -q > Dockerfile`).
- You are **comparing two IaC tools** for the same workload (`aiac
  get terraform "…"` vs `aiac get pulumi_typescript "…"`) and
  want apples-to-apples generations.
- You want a **chat REPL scoped to one artifact type** so the model
  stays on rails and does not drift into application code.

**When not to choose.**

- You need to **edit an existing IaC repo** that has its own
  conventions, module layout, and provider versions. Use `aider`
  or `claude-code` pointed at the directory — they see the
  surrounding code; aiac does not.
- You need the tool to **run `terraform plan` / `kubectl diff`**
  in the loop and react. Aiac is text-out only; use `goose` /
  `opencode` / `claude-code` with shell tools, or a purpose-built
  IaC agent.
- You need **MCP tooling** (e.g. consume a Terraform-registry MCP
  server for accurate provider docs). Aiac has no MCP client.
- You want an **agent that owns a long-running infra change**
  across many files. Try `plandex` for the planning, `aider` for
  the edits.

## Comparable tools in this catalog

- [`mods`](../mods/) — `mods 'write a Terraform module for an EKS
  cluster …' > main.tf` is the no-extra-tool answer; you lose
  aiac's post-processing and IaC-shaped system prompts but gain
  provider freedom.
- [`aider`](../aider/) — the right pick once the IaC repo exists
  and needs editing rather than scaffolding.
- [`fabric`](../fabric/) — if you want a shared, version-controlled
  prompt-pattern library for IaC generation across a team, build
  patterns there and use `fabric` instead of adding aiac.
