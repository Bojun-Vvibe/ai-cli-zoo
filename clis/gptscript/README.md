# gptscript

> Natural-language scripting language: `.gpt` files that read like prose
> but execute like programs, with tools, sub-tools, and structured I/O.
> As of the 0.x line maintained by Acorn Labs.

## TL;DR

`gptscript` treats *the prompt itself* as the source file. A `.gpt`
script is a YAML-ish header (tools, args, model, JSON schema for
output) followed by a free-form English instruction body. Running
`gptscript foo.gpt --topic kafka` parses the file, resolves declared
tools (built-ins like `sys.read`, `sys.write`, `sys.http`, or other
`.gpt` files, or OpenAPI specs, or MCP servers), runs the instruction
in an agent loop until the model produces output that satisfies the
declared schema, and prints it. Scripts can call other scripts as
sub-tools, so you compose multi-step agents the way you compose shell
functions — `script-a.gpt` lists `tools: script-b.gpt` and the
runtime wires them up.

The model is the interpreter. Your "code" is the English contract.

## Install

```bash
brew install gptscript-ai/tap/gptscript
# or
curl -sSL https://get.gptscript.ai/install.sh | sh
```

Single Go binary. `OPENAI_API_KEY` for the default provider; Anthropic,
Azure, and any OpenAI-compatible endpoint via flags or
`gptscript --default-model …`.

## One Concrete Example

`changelog.gpt`:

```
tools: sys.exec, sys.write
description: Generate a CHANGELOG.md section from git history.
args: since: the git ref to start from (e.g. v1.2.0)
output: write the result to CHANGELOG.next.md

Run `git log ${since}..HEAD --pretty=format:'%h %s'` and group the
commits into Added / Changed / Fixed / Removed sections following
Keep-a-Changelog conventions. Skip merge commits and commits whose
message starts with "chore:". Write the rendered Markdown to
CHANGELOG.next.md.
```

Run:

```
$ gptscript changelog.gpt --since v1.2.0
> Calling sys.exec: git log v1.2.0..HEAD --pretty=format:'%h %s'
> 47 commits returned, 6 chore: skipped, 3 merges skipped.
> Writing CHANGELOG.next.md (38 lines)
Done.

$ head CHANGELOG.next.md
## [Unreleased]

### Added
- `--json` flag on `report` subcommand (a1b2c3d)
- Retry-with-backoff for HTTP tool calls (e4f5g6h)
…
```

Compose: a `release.gpt` can declare `tools: changelog.gpt,
bump-version.gpt, tag-and-push.gpt` and orchestrate the three with one
English paragraph.

## Niche It Fills

**Reusable, version-controlled, composable agent scripts as
first-class artifacts.** Other agent CLIs in the catalog
(`opencode`, `codex`, `claude-code`, `aider`) are *interactive
sessions* — you tell them what to do each time. `gptscript` makes the
instruction itself the deliverable: check `.gpt` files into your repo,
review them in PRs, run them in CI. It is "shell scripts, but the
interpreter is an LLM and the syntax is English."

## Vs Already Cataloged

- **Vs [`fabric`](../fabric/):** `fabric` is a flat library of
  one-shot prompt patterns that transform stdin → stdout. `gptscript`
  scripts have args, tools, sub-tools, and a runtime loop — they can
  shell out, hit HTTP, call each other, retry. Use `fabric` for
  "summarize this transcript"; use `gptscript` for "release the
  next version."
- **Vs [`continue`](../continue/):** `continue`'s YAML config
  configures *the assistant*. A `.gpt` file *is* the assistant for
  one task. Different abstraction level: `continue` = "set up my
  general-purpose IDE helper"; `gptscript` = "ship a runnable agent
  alongside the codebase that uses it."
- **Vs [`forge`](../forge/):** `forge` defines multi-agent workflows
  in YAML with per-step model routing — closest cousin in the
  catalog. `forge` is heavier (planner/editor/reviewer roles, multi-
  model) and aimed at coding agents. `gptscript` is general-purpose
  scripting; you would write a one-page `gptscript` to fetch an API,
  reach for `forge` to build a code-modifying agent.

## Caveats

- The agent loop can drift: a vague instruction body plus a
  permissive output schema means non-deterministic results. Pin a
  `output:` JSON schema and constrain `tools:` to the smallest set
  that works.
- `sys.exec` runs commands on the host with no sandbox by default.
  Set `--workspace` and `--disable-tui-prompts=false` for human
  approval, or run inside a container if the script is not yours.
- Cost is non-obvious: a script with 5 tool calls and 3 sub-scripts
  can fan out to 20+ model calls. Use `--debug` once and read the
  trace before scheduling anything in cron.
- The `.gpt` format is still pre-1.0; expect minor schema churn
  across releases. Pin the binary version in CI.
