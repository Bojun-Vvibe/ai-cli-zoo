# sweep

> Snapshot date: 2026-04. Upstream: <https://github.com/sweepai/sweep>

Issue → PR automation. You file a GitHub issue describing what you want;
sweep opens a PR. There's a CLI but the canonical UX is "tag an issue and
walk away."

## 1. Install footprint

- **Hosted**: install the GitHub App from <https://github.com/apps/sweep-ai>.
  Zero local footprint.
- **Self-host**: `docker compose up` on the upstream repo. ~3 GB image,
  needs Postgres + Redis.
- **CLI**: `pip install sweepai` for local "issue-style" runs without a
  GitHub round-trip.

## 2. License

Apache-2.0 (core). Some hosted features are closed-source.

## 3. Models supported

OpenAI (GPT-4 family by default), Anthropic Sonnet (configurable). The
self-hosted version exposes the model choice in `config.yaml`.

## 4. MCP support

**No.** Sweep's tool surface is GitHub-shaped: search code, read files,
write files, open PR, request review.

## 5. Sub-agent model

Internal pipeline of stages, each a separate LLM call:
1. **Search** — find relevant files for the issue.
2. **Plan** — outline edits.
3. **Modify** — write the edits.
4. **Review** — self-review the diff before opening the PR.

Each stage runs once per task; not concurrent agents.

## 6. Telemetry stance

**Opt-in** for self-hosted. The hosted GitHub App obviously sees your
issue text and repo content (that's the product).

## 7. Prompt-cache strategy

Anthropic ephemeral cache when configured for Anthropic. OpenAI mode
relies on automatic prefix cache. Sweep's per-stage prompts are stable,
so cache hit rates are decent on repeated issues in the same repo.

## 8. Hot keybinds

None — there's no interactive UI. Interaction is via:

| Surface | How |
|---------|-----|
| GitHub issue | Open issue, label `sweep`, wait |
| GitHub PR | Comment `sweep: <revision request>` to iterate |
| CLI | `sweep run --issue "..."` for a local one-shot |

## 9. Killer feature, weakness, when to choose

- **Killer:** **fully async, GitHub-native flow**. The "interface" is
  literally just the issue tracker your team already uses. Nobody needs
  to learn a new tool. Triaging a backlog of small bugs becomes "label
  them all, review the PRs."
- **Weakness:** quality is bounded by the issue text. Vague issues get
  vague PRs. Also, the hosted app's free tier is small; serious use
  pushes you to self-host or pay.
- **Choose it when:** you have a backlog of well-scoped issues, the team
  already lives on GitHub, and you want to **batch-process** rather than
  sit at a TUI.
