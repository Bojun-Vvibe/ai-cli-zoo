# claude-code

> Snapshot date: 2026-04. Upstream: <https://github.com/anthropics/claude-code>

Anthropic's official terminal agent for the Claude family. Node.js TUI,
stateful project session, ReAct-style loop with native tools (file ops,
shell, web fetch, web search), hooks system, and a slash-command + skill
extension surface. Aimed squarely at "agent that lives in your repo for
days at a time."

## 1. Install footprint

- `npm i -g @anthropic-ai/claude-code` (Node 20+) or one-liner installer
  shell script. Native macOS / Linux / Windows builds also distributed.
- ~60 MB on disk after install. Bundled CLI is ESM with a small native
  bridge for raw-mode TTY handling.
- Per-project state in `.claude/` (settings, hooks, skills) plus
  per-user state in `~/.claude/` (auth, global skills, history).
- No daemon. Each `claude` invocation is either a foreground TUI session
  or a one-shot `claude -p "..."` (print) call for scripting.

## 2. License

Source-available, **not** OSI-approved. The `LICENSE` file in the public
repo is a custom Anthropic Commercial Terms reference — code is
inspectable but redistribution and derivative-CLI shipping are
restricted. Treat it as "open source for transparency, closed for
forking."

This is the single biggest license gap in this catalog: every other
entry is OSI-approved (MIT / Apache-2.0 / AGPL / FSL). If
license matters for your use case, that's a hard filter.

## 3. Models supported

Anthropic-only by design:

- Claude Opus 4 / 4.5 / 4.7 (latest large reasoning model)
- Claude Sonnet 4 / 4.5
- Claude Haiku 4 / 4.5 (cheap fast tier)
- Older 3.x family kept for backwards compat

Auth paths:

1. **Anthropic Console API key** (`ANTHROPIC_API_KEY`) — pay-as-you-go.
2. **Anthropic Pro / Max subscription login** (OAuth) — uses your
   subscription quota instead of per-token billing.
3. **Bedrock** (`CLAUDE_CODE_USE_BEDROCK=1` + AWS creds).
4. **Vertex AI** (`CLAUDE_CODE_USE_VERTEX=1` + GCP creds).

No first-class adapter for non-Anthropic providers. Routing through
LiteLLM works at the URL level but is not officially supported.

## 4. MCP support

**Yes — client only, deeply integrated.** Configure under `mcpServers`
in `~/.claude.json` (user-global) or `.mcp.json` (project-local):

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

Stdio, SSE, and streamable-HTTP transports all supported. MCP tools
appear in the same approval flow as built-in tools, with the standard
allowlist / denylist controls. Resources and prompts from MCP servers
are also exposed (resources via `@`-mention, prompts via slash
commands).

## 5. Sub-agent model

**True sub-agents.** A built-in `Task` tool spawns an isolated child
agent with its own context window, its own tool set, and a separate
tool-approval scope. Parent receives only the child's final message.
This is the most-developed sub-agent system in the catalog alongside
opencode (which copied the pattern).

You configure available sub-agents under `.claude/agents/<name>.md` —
each is a Markdown file with frontmatter for the agent's name, allowed
tools, and a system prompt. Common patterns: `explore` (read-heavy
search), `general` (multi-step research), per-project specialists.

Parallel `Task` calls are supported and the parent fans them out
concurrently, which matters for "research these 5 unrelated questions"
workloads.

## 6. Telemetry stance

**Opt-out for usage stats; explicit consent for content.** Default
behavior:

- Usage telemetry (command names, error types, token counts, no prompt
  content) sends to Anthropic. Disable with
  `DISABLE_TELEMETRY=1` env or `"telemetry": false` in settings.
- Prompt content is **not** used for training under the API and
  Pro/Max consumer subscriptions per Anthropic's published terms.
  Enterprise plans (Claude for Work) ship with zero data retention.

This is more conservative than the gemini-cli free tier (which trains
on personal-OAuth prompts by default), but more chatty than codex /
aider which require explicit opt-in for any phone-home.

## 7. Prompt-cache strategy

Uses Anthropic's **prompt caching** API aggressively. The CLI:

1. Marks the system prompt + tools schema + project context (from
   `CLAUDE.md` and skill bundles) as a `cache_control: ephemeral`
   prefix on every request.
2. Marks recent assistant turns up to a few breakpoints (Anthropic
   allows 4 cache breakpoints per request).
3. Status bar shows `cache: read N / write M tokens` on each turn so
   you can see when the prefix actually hits.

5-minute default TTL. The model's $0.10-per-1M-token cache-read price
versus the $3+-per-1M-token base price means a 30x cost cut for the
prefix on long sessions — for "8-hour pair programming on the same
repo" sessions the cache hit ratio routinely runs above 0.9.

## 8. Hot keybinds (TUI)

| Key | Action |
|-----|--------|
| `Esc` | Cancel current model call |
| `Esc Esc` (double-tap) | Edit previous message |
| `Ctrl+C` | Exit |
| `Ctrl+L` | Clear screen (keeps history) |
| `/clear` | Reset conversation |
| `/compact` | Summarize history into a shorter prefix |
| `/cost` | Show running token + dollar spend for this session |
| `/agents` | Manage sub-agent profiles |
| `/hooks` | Inspect / edit hook config |
| `/permissions` | Adjust per-tool approval rules |
| `@<path>` | Attach a file or directory to the next prompt |
| `!<cmd>` | Run a shell command directly (bypasses the model) |
| `#<text>` | Quick-add a fact to project memory (`CLAUDE.md`) |
| `Up Arrow` | Recall previous prompt |
| `Shift+Tab` | Toggle accept-edits mode |

## 9. Killer feature, weakness, when to choose

- **Killer:** **the hooks + skills + sub-agents triangle.** Hooks fire
  deterministic shell commands at lifecycle points
  (PreToolUse, PostToolUse, UserPromptSubmit, Stop, etc.) so you can
  enforce policy without trusting the model. Skills are markdown
  modules that the model can `load` on demand to inject specialized
  workflows. Sub-agents isolate context. Stack them and you can
  build a domain-specific agent on top of claude-code without
  forking the binary.
- **Weakness:** **the license + single-vendor lock-in combo.** You
  can read the source but you can't fork it into a redistributable
  product, and you can't swap to a non-Anthropic model without
  abandoning the tool. For commercial scenarios where vendor diversity
  is a hard requirement, that's two strikes.
- **Choose it when:** you're committed to the Claude family for
  reasoning quality, you want the most polished hook + sub-agent
  surface in the catalog, and the source-available license is
  acceptable for your situation.

## Pitfalls observed in real use

1. **Hook scripts run with your full shell environment.** A
   `PreToolUse` hook inherits `$PATH`, exported secrets, and the
   working directory. Treat hook scripts the same way you'd treat
   git hooks — code review them, keep them in-repo, never source
   from random gists.
2. **Sub-agents do not inherit parent context.** A `Task` call sees
   only the prompt you write into it, not the parent's accumulated
   conversation. If you skip context, the sub-agent will redo
   research the parent already finished. Pass relevant facts
   explicitly in the sub-agent prompt.
3. **`/compact` is lossy** in the same way `/compress` is in
   gemini-cli: recent turns survive verbatim, older turns become a
   summary. If a critical decision sits 50 turns back, pin it with
   `#` into `CLAUDE.md` instead of relying on `/compact` to retain it.
4. **Cache breakpoint accounting.** Only 4 `cache_control` markers
   per request. If your skills + hooks + tools schema explode the
   prefix, the CLI silently drops cache markers from the oldest
   conversation turns and your cache-read ratio collapses. Watch
   `/cost` after adding a heavy skill.
5. **`.claude/settings.local.json` is gitignored by convention** but
   the global `~/.claude.json` is not — and it can accumulate
   project paths and history across all your repos. Audit it before
   sharing logs or screenshots.
6. **Subscription auth refresh can stall the TUI** if the OAuth
   token expires mid-session — symptom is a frozen spinner with no
   error. Quit and re-run `claude` to force a refresh. API-key auth
   does not have this failure mode.
