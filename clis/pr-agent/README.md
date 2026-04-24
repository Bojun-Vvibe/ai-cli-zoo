# pr-agent

> Snapshot date: 2026-04. Upstream: <https://github.com/The-PR-Agent/pr-agent>
> Latest release at snapshot: **v0.34** (2026-04-02).
> License file: [`LICENSE`](https://github.com/The-PR-Agent/pr-agent/blob/main/LICENSE) — AGPL-3.0.

A pull-request automation CLI / bot. You point it at a PR URL and it
posts a structured **review**, a **description rewrite**, an
**`/improve` suggestions** comment, and answers ad-hoc `/ask`
questions — all as inline PR comments on GitHub, GitLab, Bitbucket,
Azure DevOps, Gitea, and Gerrit. Runs as a one-shot CLI, a long-lived
webhook server, a GitHub Action, or an inline `@pr-agent` mention bot,
all from the same Python codebase.

This entry exists because [`code-review-gpt`](../code-review-gpt/)
covers the *review* angle and [`sweep`](../sweep/) covers the
*issue-to-PR* angle, but neither is a unified **PR-time conversational
loop** with `/review`, `/describe`, `/improve`, `/ask`, `/update_changelog`,
`/add_docs`, and `/test` as first-class slash commands the team can
invoke from the PR thread.

## 1. Install footprint

- `pip install pr-agent` (Python 3.10+) for CLI mode, or pull the
  `qodoai/pr-agent:latest` Docker image for the webhook / Action /
  bot deployments.
- ~30 MB Python dependencies (LiteLLM-routed, so the heavy clients
  install lazily). Container image ~600 MB with all SCM SDKs.
- Per-repo configuration via `.pr_agent.toml` checked into the repo;
  global secrets (provider keys, SCM tokens) via env vars or a
  `secrets.toml`.
- No persistent state — each PR action is a fresh process. The bot
  deployment keeps webhook handlers in memory only.

## 2. License

AGPL-3.0 (see [`LICENSE`](https://github.com/The-PR-Agent/pr-agent/blob/main/LICENSE)).
The AGPL trigger is "running it as a network service for third
parties"; running the CLI internally or self-hosting the webhook for
your own org is fine. A separate hosted commercial offering exists
upstream — this catalog only covers the open-source CLI.

## 3. Models supported

Provider-agnostic via LiteLLM: OpenAI, Anthropic Claude, Google
Gemini / Vertex, AWS Bedrock, Azure OpenAI, Groq, DeepSeek, Mistral,
Cohere, OpenRouter, Hugging Face, Replicate, Ollama, vLLM, and any
other OpenAI-compatible endpoint via `OPENAI_API_BASE`. Per-action
model overrides (`model_review`, `model_describe`, etc.) let you spend
big on `/review` and cheap on `/describe`.

## 4. MCP support

**No.** Tool integration is via SCM REST APIs and the LiteLLM provider
list, not MCP. The agent's "tools" are the slash commands it exposes.

## 5. Sub-agent model

**Per-command pipelines, not generic sub-agents.** Each slash command
(`/review`, `/improve`, `/describe`, `/ask`, `/test`, `/add_docs`,
`/update_changelog`) is its own scripted multi-stage prompt: gather
diff → chunk by token budget → per-chunk LLM call → merge → render
to PR comment. Composition happens on the PR thread, not inside the
agent.

## 6. Telemetry stance

**Off in self-hosted CLI / webhook mode.** No analytics in the OSS
codebase by default. Egress is to (a) your configured LLM provider
and (b) the SCM REST API for the PR being acted on.

## 7. Prompt-cache strategy

**Provider-side prefix caching where supported, no client-side cache.**
The agent uses `cache_control` markers on the diff and surrounding
context blocks for Anthropic; Gemini calls go through LiteLLM's
provider-default behavior. There is no local cache database — re-
running `/review` on the same PR re-spends tokens unless the provider
deduplicates.

## 8. Hot keybinds

There is no TUI. The interface is slash commands posted to the PR:

| Command | Action |
|---------|--------|
| `/review` | Structured PR review: summary, score, suggestions, security flags |
| `/describe` | Rewrite the PR title + body from the diff |
| `/improve` | Inline code-improvement suggestions as committable comments |
| `/ask <question>` | Free-form Q&A grounded in the diff |
| `/update_changelog` | Append a CHANGELOG entry derived from the diff |
| `/add_docs` | Add docstrings / comments to changed functions |
| `/test` | Generate unit tests for changed functions |
| `/help` | List enabled commands and config |

CLI invocation mirrors the slash commands:
`pr-agent --pr_url <url> review` etc.

## 9. Killer feature, weakness, when to choose

- **Killer:** **a unified slash-command surface across every major SCM.**
  GitHub, GitLab, Bitbucket, Azure DevOps, Gitea, and Gerrit all see
  the same `/review`, `/improve`, `/ask` interface, configured by one
  `.pr_agent.toml`. No other catalog entry covers more than two SCM
  hosts; `code-review-gpt` is GitHub-only, `sweep` is GitHub-only.
- **Weakness:** **AGPL is a real constraint for some orgs.** If your
  legal team blocks AGPL-licensed dependencies — even for an internal
  bot — this entry is off the table; use `code-review-gpt` (MIT)
  instead. Also: the per-command prompt templates are opinionated; if
  your team wants a fundamentally different review rubric you are
  forking the prompts, not configuring them.
- **Choose it when:** you run more than one SCM (e.g. GitHub for OSS
  + GitLab self-hosted for internal), want one bot for both, and want
  conversational PR commands beyond a one-shot review.

## Pitfalls observed in real use

1. **The `qodoai/` Docker image is the upstream-published OSS build,
   not the hosted commercial product.** They share a name but the
   container image you pull is the AGPL codebase from this repo. If
   you need the hosted offering, that is a separate product not
   covered here.
2. **Webhook mode needs a stable public URL.** For local dev use a
   tunnel (`cloudflared`, `ngrok`); the SCM webhook delivery has no
   retries beyond the host's defaults, so an unreachable handler
   silently drops events.
3. **Rate limits bite on giant PRs.** A 5,000-line diff against
   Anthropic with `/improve` will fan out into many parallel chunks
   and trip the per-minute token limit. Set `pr_reviewer.num_max_findings`
   and the chunking caps in `.pr_agent.toml` *before* turning the bot
   loose on a busy repo.
4. **The bot needs write access to comment.** Inline-suggestion
   comments require `pull-requests: write` (GitHub) or `Developer`
   role (GitLab). Read-only PATs silently produce 403s the user only
   sees in the bot's stderr.
5. **Slash commands are case-sensitive and must start the line.**
   `Please /review this` is ignored; `/review please` is honored.
   The bot does not parse natural language around the command.
