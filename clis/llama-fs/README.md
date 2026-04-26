# llama-fs

> Snapshot date: 2026-04. Upstream: <https://github.com/iyaja/llama-fs>

A **self-organizing filesystem agent**: point `llama-fs` at a
messy directory (the `~/Downloads` graveyard of `IMG_4419.jpg`,
`Untitled.pdf`, `meeting-notes-final-v2.docx`, `screenshot 2024
... .png`), and it inspects each file's contents — OCR for
images, PDF text extraction, document parsing — asks an LLM to
propose a semantic name and a target folder, then either renames
in place or watches the directory and reorganizes new files as
they land. The agent loop is one model call per file (or per
batch) plus deterministic mv operations.

It is the catalog's reference for **a content-aware
filesystem-edit agent CLI** — narrow scope, single-purpose,
contrasted against general coding agents that happen to be able
to rename files.

## 1. Install footprint

- `git clone https://github.com/iyaja/llama-fs && cd llama-fs &&
  pip install -r requirements.txt` (Python 3.10+; no PyPI
  package).
- Pulls `groq`, `ollama`, `pillow`, `pypdf2`, `python-docx`,
  `watchdog`, `fastapi`, `agentops`. ~250 MB venv.
- Models: Groq-hosted Llama 3 (default, fastest free tier),
  Ollama for fully local (`OLLAMA_MODEL=llama3` env var), any
  OpenAI-compatible endpoint by editing the client wiring.
- Optional: a separate Electron client lives in `electron-react/`
  for a GUI on top of the same FastAPI backend; the CLI surface
  is `python -m server` + curl, or the `cli/` script.

## 2. Repo, version, license

- Repo: <https://github.com/iyaja/llama-fs>
- Version checked: **unversioned** — no tagged releases. HEAD is
  the canonical "latest" reference, last meaningful commit
  2025-08.
- HEAD pinned at this snapshot:
  `0a693717ca1845b3ae8c208b2929d987fcdabb81`.
- License: MIT. License file at
  [`LICENSE`](https://github.com/iyaja/llama-fs/blob/main/LICENSE).

## 3. What it actually does

The agent has three modes:

**Batch mode.** `POST /batch` with a directory path. The server
walks the tree, reads each file (mime-aware: text → raw,
PDF → `pypdf2` extraction, DOCX → `python-docx`, image → base64
to vision model or OCR fallback), batches up to N files into a
single LLM prompt of the form *"Here are file summaries. Propose
new names and a folder hierarchy that groups related ones."*,
and returns a JSON list of `{src, dst}` moves. The CLI / GUI
shows the proposed plan; the human approves or edits before
anything moves.

**Watch mode.** `POST /watch` registers a `watchdog` observer on
a directory. As new files land (a downloaded receipt, an
AirDropped photo), the agent classifies the single file in
isolation against the existing folder structure and proposes one
move. Lower latency, no cross-file batching.

**Commit mode.** `POST /commit` with the approved plan executes
`shutil.move` on each pair, optionally with an undo log.

The model prompt is the interesting part: rather than asking
"rename this file", the prompt asks the model to act as a
**filesystem-organizing librarian** producing a flat JSON of
proposed paths, with the file's extracted contents inlined. The
model never executes mv — it returns text that the deterministic
host code parses and applies, classic "LLM-as-planner,
host-as-actuator" separation.

## 4. MCP support

None. The agent surface is HTTP / FastAPI; an MCP wrapper would
be a small adapter that re-exposes `/batch` and `/commit` as MCP
tools, but none ships.

## 5. Sub-agent model

None — flat single-agent loop per file or per batch. No
delegation, no planner / executor split beyond the LLM-vs-host
boundary described above.

## 6. Telemetry stance

The shipped code wires `agentops.init(...)` for trace capture if
the env var is set; default config emits to `agentops.ai` only
if the user provides an API key, otherwise nothing is sent. The
default model backend (Groq) sees every file's extracted
contents — for sensitive directories, swap to Ollama via
`OLLAMA_MODEL=llama3` and the loop runs entirely local.

## 7. Token / context strategy

Bounded per file: each file's extracted text is truncated to a
budget (~2k tokens) before being sent. Batch mode sums these
into one prompt, so very large directories are chunked into
multiple LLM calls. The agent does not maintain a cross-call
memory — each batch is stateless, which is also why repeated
runs over the same directory can produce slightly different
folder names unless temperature is pinned.

## 8. Hot keybinds

No CLI TUI of its own — the FastAPI server is curl- or
GUI-driven. The Electron client adds drag-to-approve / undo
buttons; the headless workflow is `curl -X POST :8000/batch -d
'{"path":"~/Downloads"}'` → review JSON → `curl -X POST
:8000/commit`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A focused, content-aware mv-only agent
that the host-side approval step keeps safe.** Unlike a coding
agent that could be told "organize my Downloads folder" and
might helpfully invent shell pipelines that delete things,
`llama-fs` only ever proposes `(src, dst)` pairs — the action
surface is one verb. The watch-mode daemon converts the agent
into background plumbing: nothing in `~/Downloads` stays as
`IMG_4419.jpg` for more than a few seconds before being filed
under `2026/04/photos/`.

**Weakness.** Unmaintained-ish: no tagged releases, no PyPI
package, last commit several months stale, dependencies pinned
to mid-2025 versions. The Groq default ships your filenames + a
truncated peek of every file off-machine; for any private
content, manual swap to Ollama is required and not all
extraction paths are covered (vision-model image classification
needs the cloud path). No conflict resolution if two files map
to the same proposed name — the second move overwrites silently
unless caught at review time. No dry-run flag at the directory
level beyond "look at the JSON before calling commit".

**Choose `llama-fs` when** the workload is "I have a hoard of
unstructured files and want a one-shot or daemonized renamer
that thinks about contents, not just extensions". **Choose
something else when** you want a maintained product
(commercial: Hazel; open: try a generic
[`open-interpreter`](../open-interpreter/) one-shot prompt for
ad-hoc reorgs), when filesystem changes need an audit trail
and rollback (write a small script that wraps `git mv` in a
worktree), or when the files are code and the right action is
"refactor them" rather than "rename them" (use
[`aider`](../aider/) / [`claude-code`](../claude-code/) /
[`opencode`](../opencode/)).
