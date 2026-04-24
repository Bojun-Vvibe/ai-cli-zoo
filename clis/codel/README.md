# codel

> Snapshot date: 2026-04. Upstream: <https://github.com/semanser/codel>
> Last verified version: **0.2.2** (released 2024-04-05; still the
> latest tag as of 2026-04). License file:
> [`LICENSE`](https://github.com/semanser/codel/blob/main/LICENSE) —
> AGPL-3.0.

A self-hosted, Docker-only autonomous agent. You bring it up with
`docker run`, open `localhost:3000` in a browser, and type a goal;
the agent then runs commands inside its **own ephemeral Docker
sidecar**, browses the web with a built-in headless browser, and
edits files in a built-in code viewer. Everything — every shell
command, every command output, every browser page, every file edit —
is journaled to a PostgreSQL database that travels in the same
container.

It is one of the few catalog entries that is **a fully bundled agent
runtime**: you do not install a CLI on your host; you install a
self-contained system that *includes* the sandbox.

## 1. Install footprint

The shipping path is one container:

```
docker run \
  -e OPEN_AI_KEY=... \
  -e OPEN_AI_MODEL=gpt-4-0125-preview \
  -p 3000:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/semanser/codel:latest
```

Notes that matter:

- **Mounts the host Docker socket.** This is how codel spawns its
  per-task worker containers. Anything that can talk to the codel
  container can therefore launch arbitrary containers on the host —
  treat it as root-equivalent and do not expose port 3000 beyond
  `127.0.0.1`.
- Bundles its own PostgreSQL inside the image; nothing to install
  separately.
- Bundles the headless browser (built on `go-rod`); no Chrome /
  Chromium install on the host.
- Frontend is a SvelteKit web app served from the same container.
- No native macOS / Linux / Windows binary — Docker is the only
  supported install path.

## 2. Repo, version, license

- Repo: <https://github.com/semanser/codel>
- Version checked: **0.2.2** (2024-04-05). Project has been quiet
  since 2024 and 0.2.2 is still the most recent tag in 2026-04.
- License: AGPL-3.0. License file at the repo root:
  [`LICENSE`](https://github.com/semanser/codel/blob/main/LICENSE).
- Container image: `ghcr.io/semanser/codel:latest`.

The AGPL is significant: any modification of codel that is exposed
over a network (which is the *only* way codel runs) triggers the AGPL
network-use clause. If you fork codel for an internal team the source
of your fork has to be available to users of that internal instance.

## 3. What it actually does

The supervisor agent runs a perceive → plan → act loop:

1. Receives a task from the web UI.
2. Picks a Docker image appropriate to the task (the README calls
   this "automatic Docker-image picker" — e.g. `node:20`,
   `python:3.12`, `golang:1.22`).
3. Spawns a worker container from that image with the host workspace
   mounted in.
4. Inside the worker: runs shell commands, edits files, and — when
   the plan calls for it — drives the built-in headless browser to
   read documentation / Stack Overflow / package READMEs.
5. Persists every command, every output, every file diff, every page
   URL to PostgreSQL so the web UI can replay the run step by step.

The user interface is **the web app, not a CLI**. There is no
`codel` binary you run interactively; you watch the agent work in a
browser and intervene by typing follow-up messages.

## 4. MCP support

None. The tool surface (shell, browser, editor) is hard-coded in
the Go backend. Codel does not consume MCP servers and does not
expose one.

## 5. Sub-agent model

Single supervisor agent that spawns **per-task worker containers**.
The "sub-agent" here is a sibling Docker container running the same
agent binary under a fresh task scope, not a separate model role.
There is no planner / coder / reviewer split inside one task.

## 6. Telemetry stance

Off (no analytics in the container itself). All egress is the
configured LLM provider and whatever URLs the headless browser is
told to visit. The PostgreSQL journal stays inside the container's
volume; nothing is shipped upstream by codel.

Two non-obvious egress sources:

- The Docker-image picker pulls images from Docker Hub (or whatever
  registry the host's daemon is configured to use) on first task.
- The headless browser will fetch arbitrary URLs the model decides
  to visit — including pages with tracking pixels.

## 7. Token / context strategy

The journal in PostgreSQL grows monotonically per task. The agent
window passed to the model is a sliding tail: recent
command/output/diff entries plus the original task statement.
There is no semantic compression; very long tasks degrade gracefully
into "the model only remembers the last N steps" rather than into a
hard error.

## 8. Hot keybinds

None — the interface is a web UI in your browser, not a TUI.

## 9. Killer feature, weakness, when to choose

**Killer feature.** It is the only catalog entry that ships a
**bundled multi-modal agent runtime** (LLM + sandbox + browser +
editor + journal) as a single container. Compare to
[`OpenHands`](../openhands/) which has the same ambition but is a
heavier install with more moving parts; codel is closer to "one
docker run" than anything else with this scope.

**Weakness.** Two big ones:

1. **Project is dormant.** 0.2.2 from 2024 is still current in
   2026-04. The model defaults (`gpt-4-0125-preview`) are stale; you
   will need to override `OPEN_AI_MODEL` to anything modern. The
   browser and editor stacks are not getting security fixes.
2. **The Docker socket mount is a sharp edge.** A model that decides
   to `docker run --privileged ... -v /:/host` has root on the host.
   This is the trade-off for "agent-spawns-its-own-sandbox" UX, and
   it is fine for a personal laptop but unacceptable for any shared
   machine.

Other limits: web-UI-only (no SSH-friendly mode), no MCP, no
multi-provider routing beyond OpenAI / Ollama, no telemetry / cost
tracking, no resume across container restarts unless you mount the
PostgreSQL volume.

**When to choose.**
- You want to **try** the "autonomous agent that runs in its own
  Docker sandbox" pattern with the lowest possible install cost.
- You explicitly want a **browser-using** agent (most catalog
  entries do not have a browser tool) and you do not want to wire
  Playwright yourself.
- You want a **journal-replayable** record of every step the agent
  took, with diffs and command outputs, that survives the agent's
  death.

**When to skip.**
- You are on a shared machine where you cannot give the agent
  unrestricted Docker access.
- You need a maintained project — pick
  [`OpenHands`](../openhands/) instead; same shape, active.
- You want a CLI you can pipe into other CLIs — codel is a web app.
- You need MCP, sub-agents, or multi-provider routing — see
  [`opencode`](../opencode/), [`goose`](../goose/), or
  [`forge`](../forge/).

## 10. Compared to neighbors in the catalog

| Tool | Surface | Sandbox model | Browser tool | Journal / replay | Active |
|------|---------|---------------|--------------|-----------------|--------|
| codel | Web UI in container | Spawns Docker workers (host socket) | Built-in headless | PostgreSQL bundled | Dormant since 2024 |
| [OpenHands](../openhands/) | CLI + optional web | Linux sandbox per task | Built-in browser agent | Event store on disk | Active |
| [openhands → CLI mode](../openhands/) | Terminal | Same as above | Same as above | Same as above | Active |
| [opencode](../opencode/) | TUI / IDE | Host shell (you trust) | No (MCP can add) | Session log on disk | Active |
| [aider](../aider/) | REPL | Host shell (you trust) | No | Per-commit history (git) | Active |

Decision shortcut:

- "I want the lightest possible 'agent in a sandbox' demo I can boot
  in five minutes" → `codel` (with the dormancy warning).
- "I want the same shape but maintained" → `OpenHands`.
- "I want a CLI I can compose with other CLIs" → `opencode` /
  `aider` / `claude-code`.
