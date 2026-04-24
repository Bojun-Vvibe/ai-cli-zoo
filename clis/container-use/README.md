# container-use

> Snapshot date: 2026-04. Upstream: <https://github.com/dagger/container-use>.
> Pinned version: **v0.4.2** (2025-08-19).

A containerized sandbox layer that lets multiple coding agents work in parallel
on the same repo without stepping on each other. Not an agent itself — it's the
isolation primitive that other agents in this catalog plug into via MCP.

## 1. Install footprint

- `brew install dagger/tap/container-use` (macOS, recommended) or
  `curl -fsSL https://raw.githubusercontent.com/dagger/container-use/main/install.sh | bash`
  (Linux / macOS).
- Single static Go binary, ships as both `container-use` and `cu`.
- Hard runtime dep: a working Docker / Podman daemon, plus
  [Dagger](https://dagger.io) (auto-pulled on first run).
- No long-lived server of its own — each agent session spawns its own container.

## 2. License

Apache-2.0 — `LICENSE` at the repo root
(<https://github.com/dagger/container-use/blob/main/LICENSE>).

## 3. Models supported

None directly. Container-use is model-agnostic; it sits between *whatever*
agent you're running and the filesystem / shell. The agent picks the model.

## 4. MCP support

**Yes — and MCP is the entire interface.** `container-use stdio` is an MCP
server you mount inside any MCP-capable client
([opencode](../opencode/), [claude-code](../claude-code/), [crush](../crush/),
[goose](../goose/), [cline](../cline/), Cursor, …). The exposed tools let the
agent create an environment, run commands inside it, edit files inside it,
inspect logs, and check out the resulting git branch.

## 5. Sub-agent model

None of its own. Its purpose is to make *other* agents safe to run
**concurrently**: each agent session lands in a fresh container on its own git
branch, so N parallel attempts at the same task don't corrupt each other or
your working tree.

## 6. Telemetry stance

**Off.** No analytics in the binary. Egress is the Dagger engine pulling
container images from the registries you've configured.

## 7. Prompt-cache strategy

N/A — container-use never talks to a model. Caching is a property of whichever
agent client mounts it.

## 8. Hot keybinds

Not a TUI — a CLI + MCP server. Useful subcommands:

| Command | Action |
|---------|--------|
| `cu stdio` | Run as MCP server over stdio |
| `cu list` | List active agent environments |
| `cu log <env>` | Stream the command + output history of an env |
| `cu terminal <env>` | Drop into the agent's container shell to debug |
| `cu watch` | Live-tail every running agent |
| `git checkout <env-branch>` | Review the agent's work as ordinary git |

## 9. Killer feature, weakness, when to choose

**Killer feature.** Per-agent **container + git-branch** isolation exposed as
plain MCP tools. You can launch three agents at the same prompt, watch them
diverge, and `git diff` the winners — without rolling your own Docker plumbing
or worrying that two agents will both `rm -rf` the same `node_modules`.

**Weakness.** Marked "experimental" upstream; the tool surface and CLI flags
have shifted between minor versions, and the agent-side rules file
(`agent.md`) needs a refresh whenever upstream renames a tool.

**When to choose.** You've adopted any of the MCP-capable agents in this
catalog and want to (a) run several in parallel safely, (b) keep the host
filesystem untouched until you accept a branch, or (c) get a real audit log
of every shell command an agent ran. Pair with [opencode](../opencode/) or
[claude-code](../claude-code/) for the most polished experience.
