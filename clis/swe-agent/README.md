# swe-agent

> Snapshot date: 2026-04. Upstream: <https://github.com/SWE-agent/SWE-agent>.
> Pinned version: **v1.1.0** (2025-05-22).

A research-grade autonomous agent from Princeton / Stanford that takes a
GitHub issue (or any natural-language task) and edits a repo until the task
is solved. Originally built as the reference implementation for SWE-bench;
the catalog's most academically grounded agent.

## 1. Install footprint

- `pip install sweagent` (Python 3.11+) or `pipx install sweagent`.
- Uses Docker for its sandbox by default; the `swerex` runner pulls a
  per-task image so the agent's `bash`, `edit`, and `submit` tools never
  touch your host shell.
- ~150 MB of Python deps once tree-sitter, Pydantic, and litellm land.
- macOS / Linux first-class; Windows works via WSL.

## 2. License

MIT — `LICENSE` at the repo root
(<https://github.com/SWE-agent/SWE-agent/blob/main/LICENSE>).

## 3. Models supported

Anything `litellm` routes — OpenAI (GPT-4o, o-series), Anthropic (Sonnet,
Opus), Gemini, Bedrock, Vertex, Together, Groq, DeepSeek, Ollama,
llama.cpp, and any OpenAI-compatible HTTP endpoint. The agent loop is
model-agnostic; the tuned prompts target frontier reasoning models.

## 4. MCP support

**No.** SWE-agent's tool surface is a hand-curated set of bash + file-edit
"commands" defined in YAML, designed for reproducibility on benchmarks
rather than plug-and-play with arbitrary MCP servers.

## 5. Sub-agent model

None — single agent loop. Parallelism happens at the *task* layer: `sweagent
run-batch` fans out one independent agent instance per SWE-bench problem.

## 6. Telemetry stance

**Off.** No analytics in the binary. Egress is the configured LLM provider
plus whatever the sandboxed `bash` calls (e.g. `pip install`) reach out to.
Per-task trajectories are written to local JSON for offline analysis.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on the system prompt + scrolling history window.
The trajectory is intentionally short: SWE-agent uses a "demonstration +
recent N turns" window so cache hit rates stay high across long runs.

## 8. Hot keybinds

Not a TUI. The CLI surface is task-shaped:

| Command | Action |
|---------|--------|
| `sweagent run --config <yaml> --problem-statement-path <md>` | Solve one task |
| `sweagent run-batch --instances <jsonl>` | Fan out across many tasks (e.g. SWE-bench) |
| `sweagent inspect <traj.json>` | Replay a saved agent trajectory |
| `sweagent inspector` | Open the local web inspector for trajectory diffing |

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Reproducible benchmark-grade trajectories.** Every
agent run emits a fully-typed JSON of every observation, action, and tool
output, replayable in `sweagent inspect`. The whole agent — prompts, tools,
parser, demonstrations — is one YAML file you can diff and version. Nothing
else in the catalog is this rigorous about reproducibility.

**Weakness.** Upstream is steering users toward
[mini-SWE-agent](https://github.com/SWE-agent/mini-SWE-agent) for new work
("matches the performance, much simpler"). SWE-agent 1.x will keep getting
bug-fixed, but the research focus has moved. Also: configuration via YAML
is powerful but verbose — there's no `--quick` story.

**When to choose.** You're (a) doing agent research and need a known-good
baseline that posts SoTA-class numbers on SWE-bench, (b) running large
fan-out evals where reproducibility beats UX, or (c) studying agent-loop
design and want a hackable reference. For day-to-day "fix this bug in my
repo" work, [aider](../aider/) or [opencode](../opencode/) will feel
lighter. SWE-agent also ships an "EnIGMA" mode for academic CTF / security
benchmarks; that surface is research-only and out of scope for production
use.
