# plandex

> Snapshot date: 2026-04. Upstream: <https://github.com/plandex-ai/plandex>

A terminal AI agent built around an explicit **plan** abstraction. You don't
chat with the model — you build a plan, branch it, apply it, roll it back.

## 1. Install footprint

- One-line install: `curl -sL https://plandex.ai/install.sh | bash`.
- ~30 MB Go binary (`plandex` / `pdx`).
- Requires a backend server. Two modes: **local server** (auto-spawned, free,
  uses your API keys) or **Plandex Cloud** (managed, paid, optional).
- macOS, Linux. Windows via WSL.

## 2. License

Dual: **AGPL-3.0** for the open-source core, MIT for the CLI client.
The AGPL on the server matters if you redistribute it; for personal use,
no obligations.

## 3. Models supported

- Anthropic Sonnet/Opus
- OpenAI o-series, GPT-4.1/5
- Configurable per-role (planner, coder, summarizer, etc.) — you can use
  Sonnet for planning and a cheap model for summarization, for example.

## 4. MCP support

**No.** Plandex pre-dates MCP and hasn't adopted it. Tool surface is
fixed: file read/write, shell, git.

## 5. Sub-agent model

Internal **planner / builder / verifier** split — same conversation
shares state, but each role is a separate LLM call with its own system
prompt. Closer to "pipeline" than "sub-agent tree".

## 6. Telemetry stance

**Opt-in.** Local server sends nothing by default. Cloud mode obviously
sees your prompts (it's running them).

## 7. Prompt-cache strategy

Anthropic ephemeral cache on system prompt + plan context. Plandex's
plan-snapshot model means each "build" step replays a similar prefix,
so cache hit rate is unusually high for long plans.

## 8. Hot keybinds (REPL + subcommands)

Plandex is mostly subcommands, not a TUI. Common ones:

| Command | Action |
|---------|--------|
| `pdx new` | Start a new plan |
| `pdx tell "..."` | Add instruction to current plan |
| `pdx load <files>` | Add files to plan context |
| `pdx tree` | Show plan branch tree |
| `pdx checkout <branch>` | Switch plan branch |
| `pdx apply` | Apply pending changes to filesystem |
| `pdx reject` | Discard pending changes |
| `pdx convo` | Open conversation log |
| `pdx diff` | Show pending diff |

## 9. Killer feature, weakness, when to choose

- **Killer:** **plan branching with rollback**. You can fork a plan, try
  two approaches in parallel, diff them, throw one away. No other tool
  here treats long-horizon work as a first-class versioned object.
- **Weakness:** AGPL on the server scares some orgs off. The "plan" mental
  model is a learning curve — first session feels heavyweight vs `aider`'s
  "just type and go".
- **Choose it when:** the task is genuinely multi-hour ("port this service
  from Express to Fastify, then update tests, then write a migration
  guide") and you want versioned, branchable progress instead of one
  unbroken chat scroll.
