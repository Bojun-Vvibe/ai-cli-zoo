# codebuff

> Snapshot date: 2026-04. Upstream: <https://github.com/CodebuffAI/codebuff>.
> Pinned version: **v1.0.644** (npm `codebuff@1.0.644`; matching git tag
> `v1.0.644`).

A terminal coding agent built around an internal **multi-agent pipeline**:
a File Picker, a Planner, an Editor, and a Reviewer are distinct LLM roles
that hand work to each other on every task. The shape competes with
[claude-code](../claude-code/) and [opencode](../opencode/) but the
multi-role split is exposed as the headline feature.

## 1. Install footprint

- `npm install -g codebuff` (Node 18+). Also published as `freebuff` (the
  free, ad-supported tier) under the same binary surface.
- Single Node CLI, ~30 MB on disk after the postinstall step pulls
  platform-native deps.
- macOS / Linux / Windows. State lives in `~/.codebuff/` and a per-repo
  `knowledge.md` that the agent maintains itself.

## 2. License

Apache-2.0 — `LICENSE` at the repo root
(<https://github.com/CodebuffAI/codebuff/blob/main/LICENSE>). The npm
manifest historically advertised `MIT`; the upstream LICENSE file is the
source of truth and is Apache-2.0 as of v1.0.644.

## 3. Models supported

Routed through Codebuff's hosted backend by default; the backend
multiplexes Anthropic Claude (Sonnet / Opus / Haiku), OpenAI GPT-4o and
o-series, and Google Gemini for the different sub-agent roles. Self-hosted
keys can be supplied for direct provider calls; per-role model assignment
is the differentiator versus single-model agents.

## 4. MCP support

**Yes (client).** Custom agents declared in `.agents/` can mount MCP
servers as tools. The default agent set does not depend on MCP — the
built-in tools cover file edit / read / shell / search natively.

## 5. Sub-agent model

**Multi-role pipeline as the product.** The default flow spawns:

1. **File Picker** — scans the repo, returns a relevance-ranked file list.
2. **Planner** — turns the user request + file list into an ordered edit plan.
3. **Editor** — applies the plan with diff-shaped tool calls.
4. **Reviewer** — validates the edits, optionally re-runs steps.

Custom agents are TypeScript files in `.agents/types/` that declare which
tools they have access to and which other agents they may spawn — a
stricter contract than the free-form sub-agent spawning in
[claude-code](../claude-code/).

## 6. Telemetry stance

**On (hosted backend).** Default routing goes through Codebuff's servers,
which observe prompts and emit analytics for billing and product metrics.
Running with self-hosted provider keys reduces this surface to "the
provider sees your prompts" but does not fully eliminate the hosted
control plane in v1.0.x. Treat it as **opt-in cloud agent**, not a
local-only tool.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on the system prompt and the scrolling per-role
history. Because each sub-agent has its own conversation, cache hit rates
are computed per-role rather than across the whole task — which keeps the
Planner's long context cheap even when the Editor's window churns.

## 8. Hot keybinds (REPL)

Codebuff is a readline REPL. Slash commands drive the multi-agent surface:

| Command | Action |
|---------|--------|
| `/init` | Generate `knowledge.md` + `.agents/` scaffolding for this repo |
| `/agents` | List available agents (built-in + custom from `.agents/`) |
| `/run <agent> <task>` | Invoke a specific agent directly, bypassing the default pipeline |
| `/diff` | Show pending edits before accept |
| `/undo` | Revert the most recent agent edit batch |
| `/cost` | Per-role token + USD spend for the current session |

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A multi-agent pipeline you can rewrite.** Other
catalog tools either ship one model + one loop ([aider](../aider/),
[mods](../mods/)) or expose sub-agents as an opaque escape hatch
([claude-code](../claude-code/)'s `Task`). Codebuff makes the Planner /
Editor / Reviewer split first-class, declared in TypeScript files you
check into your repo, with explicit tool allowlists per role and explicit
"this agent may spawn that agent" edges.

**Weakness.** The default code path is hosted SaaS — telemetry is on,
your prompts traverse Codebuff's servers, and self-hosting takes effort.
The "ad-supported `freebuff`" variant trades injected ads in the REPL for
a no-credit-card experience, which some teams will not stomach. Also: at
v1.0.644 the project releases multiple times a day; pin a version in CI.

**When to choose.** You want claude-code-shaped ergonomics but believe
the *right* abstraction is a typed multi-role pipeline, not one heroic
agent with sub-agent escape hatches. Strong fit for teams who are
already thinking in Planner / Editor / Reviewer terms and want that
separation enforced by the framework. Avoid if local-only or
zero-telemetry is a hard requirement — reach for [aider](../aider/) or
[opencode](../opencode/) instead.
