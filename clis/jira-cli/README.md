# jira-cli

- **Repo:** https://github.com/ankitpokhrel/jira-cli
- **Version:** v1.7.0 (2025 release line)
- **License:** MIT ([LICENSE](https://github.com/ankitpokhrel/jira-cli/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install ankitpokhrel/jira-cli/jira-cli` · `go install github.com/ankitpokhrel/jira-cli/cmd/jira@latest` · prebuilt binaries on the GitHub releases page · Docker `ghcr.io/ankitpokhrel/jira-cli`

## What it does

`jira-cli` is a single static Go binary that turns the Atlassian
Jira Cloud / Data Center REST API into a fast, scriptable, and
TUI-driven command line. After a one-time `jira init` (server
URL + email + API token, or PAT for on-prem), every common Jira
verb has a flag-driven invocation: `jira issue list -a$(jira me)
--plain` dumps your assigned issues as TSV for piping; `jira
issue create -tBug -s"summary" -b"body" --no-input` is a
non-interactive create suitable for CI; `jira sprint list
--current --plain` prints the active sprint's issue keys; `jira
issue move PROJ-123 "In Review"` walks the workflow transition;
`jira issue link PROJ-123 PROJ-456 "Blocks"` adds typed links;
`jira issue assign PROJ-123 $(jira user search --query "Alice"
--plain | head -1)` reassigns. The interactive TUI (`jira issue
list` without `--plain`) renders a keyboard-driven board with
filter / sort / view / comment / transition hotkeys, and JQL
power users can drop straight into raw JQL with `jira issue list
--jql "project = X AND status != Done AND updated >= -7d"` —
results stream as TSV, JSON, plain table, or a rendered TUI
depending on flags. Configuration is per-project (`.jira/.config.yml`
in the repo root) so switching repos auto-switches the active
Jira project / board / sprint context, and the binary speaks
both Cloud and Server / Data Center via the same flags.

## What's interesting

- **TUI + scriptable in one binary** — same command renders an
  interactive board (`jira issue list`) or pipes TSV
  (`--plain`) for shell composition; no separate "library mode"
  to learn.
- **Per-project config via `.jira/.config.yml`** — switching
  repos auto-switches active project / board / sprint context,
  so multi-project engineers don't `--project` every command.
- **JQL escape hatch** — `--jql` accepts arbitrary JQL strings,
  so anything the web UI's advanced search can express is one
  flag away (no need to learn a parallel filter DSL).
- **Sprint + epic-aware** — first-class `jira sprint`, `jira
  epic`, and `jira issue link` commands cover the agile
  metadata that bare REST clients leave to you to compose.
- **Cloud + Server / Data Center** — same CLI talks to
  Atlassian's hosted Jira and to self-hosted on-prem instances
  via PAT, so a team straddling both deployments has one tool.

## AI-native angle

`jira-cli` is the missing structured-output adapter between an
LLM agent and a real ticket tracker. An agent that can call
`jira issue list --jql "assignee = currentUser() AND status =
\"To Do\"" --plain` reads back its own backlog as parseable TSV,
plans against it, and posts updates with `jira issue comment
add PROJ-123 "patch posted at <pr-url>"` — all without parsing
HTML or maintaining a Jira REST wrapper inside the agent. For
"close the loop" workflows (LLM pair-programmer opens a PR →
links the PR to the originating ticket → transitions the ticket
to In Review → comments the diff summary), the four shell
invocations are stable, idempotent, and shippable as a tool
manifest to any agent framework that exposes `Bash` /
`shell.exec`. The `--plain` mode's TSV output is small enough
to fit cleanly inside a tool-call response window, which makes
`jira-cli` a natural counterpart to [`gh`](../gh/) for ticket /
PR cross-linking from inside an autonomous loop.