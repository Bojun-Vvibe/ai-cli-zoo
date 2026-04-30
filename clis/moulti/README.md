# moulti

> **A TUI that turns a long, structured shell
> script run into a collapsible multi-step
> dashboard with live, scrollable per-step
> output, statuses, and timings** — a Python /
> Textual app (`moulti run …` server +
> `moulti step add …` client commands) that
> takes the noisy 30-minute log of a CI-shaped
> bash script and renders it as a folder of
> named, color-coded steps you can collapse,
> re-open, and re-read after the fact. Pinned
> to **v1.34.1**
> ([LICENSE](https://github.com/xavierog/moulti/blob/master/LICENSE),
> MIT).

Source: <https://github.com/xavierog/moulti>

## TL;DR

`moulti` decomposes a single big shell script
into **named "steps"** — each step is its own
scrollable, collapsible log pane with a status
(`pending` / `in_progress` / `success` /
`warning` / `error` / `inactive`), an icon, a
color, and a timer. The model is a tiny
client/server: `moulti run` (or `moulti
init`) starts a Textual TUI server bound to
a Unix socket, and your script invokes
`moulti step add my_step --title "Build"`
followed by `command | moulti pass my_step`
to stream the command's stdout into that
step's pane. You can update the title /
status / classes mid-flight (`moulti step
update my_step --classes success`), append
arbitrary text (`moulti step append my_step
"…done"`), open a question modal that blocks
the script until the human picks an answer
(`moulti question add …`), and group steps
into nested **`divider` / `step` /
`buttonquestion` / `inputquestion`**
hierarchies. Output is **searchable**
(`/`-style filter across all steps), the TUI
keeps a **scrollback per step** independent
of terminal size, and a `--save` flag on
shutdown writes the full session
(steps + output + statuses + timings) to a
single `.moulti` archive that `moulti load
that.moulti` re-opens later for post-mortem.
For long ops it's the difference between
"4000 lines of mixed `apt`, `make`, and
`pytest` output streaming past" and "five
collapsed sections, one of which is red,
expand it to see why".

## Install

```bash
# pipx (recommended — isolates the Textual deps)
pipx install moulti

# pip
pip install --user moulti

# from source
git clone https://github.com/xavierog/moulti
cd moulti && pip install .

# verify
moulti --version    # 1.34.1
```

## Basic usage

```bash
# 1) start the TUI server
moulti init &

# 2) from your script (same shell or any subshell that
#    inherits MOULTI_SOCKET_PATH), declare and stream:
moulti step add build --title "Build the project"
make 2>&1 | moulti pass build
moulti step update build --classes success

moulti step add tests --title "Run tests"
pytest -q 2>&1 | moulti pass tests \
  && moulti step update tests --classes success \
  || moulti step update tests --classes error

# 3) ask the human something mid-script
moulti buttonquestion add deploy_q \
  --title "Deploy to staging?" \
  --button yes success deploy_yes \
  --button no  error   deploy_no
answer=$(moulti buttonquestion get-answer deploy_q)

# 4) save the whole session for later review
moulti save run-$(date +%Y%m%d).moulti
```

One-shot wrapper: `moulti run -- ./your-script.sh` boots
the TUI, runs the script with `MOULTI_SOCKET_PATH` set,
and exits when it finishes.

## When to choose

- **Your "one big script" produces hundreds
  to thousands of lines of mixed-stage
  output** (provisioning, build matrices, big
  test suites, dotfile bootstraps, ML
  training pipelines) — `moulti` gives you
  one collapsible block per stage, with a
  status colour at a glance.
- **You want a CI-build-page experience for
  scripts that don't run in CI** — local
  `make all`, dev-environment bootstraps,
  release-checklist scripts; render them
  like a GitHub Actions / Jenkins job page
  in your own terminal.
- **You need to ask the human a question
  mid-script without breaking the flow of
  visible output** — `buttonquestion` /
  `inputquestion` blocks the script and
  surfaces the prompt as a modal in the same
  TUI, then resumes streaming.
- **You want a saveable, replayable record**
  of a long run for after-the-fact debugging
  — the `.moulti` archive captures every
  step's output + status + timing, no log
  parsing required.

## When NOT to choose

- **Your script is short and linear** —
  `moulti` is overhead for a 20-second build;
  use [`tailspin`](../tailspin/) or plain
  output.
- **You need a generic process / job
  dashboard** (run N independent commands in
  parallel, watch them all) — use
  [`mprocs`](../mprocs/) or
  [`pueue`](../pueue/); `moulti` is
  step-oriented around one driving script,
  not a process pool.
- **You want a follow-only log viewer with
  syntax highlighting for arbitrary log
  files** — use [`toolong`](../toolong/) or
  [`lnav`](../lnav/); `moulti` requires the
  producing script to cooperate via the
  client commands.

## Why it fits the zoo

Joins the "structured terminal output for
long-running work" cluster alongside
[`mprocs`](../mprocs/) (parallel processes,
one pane each), [`tailspin`](../tailspin/)
(syntax-highlighted log follower),
[`toolong`](../toolong/) (multi-file log
TUI), and [`pueue`](../pueue/) (job queue
with output capture). `moulti`'s specific
gap is **a single driving script + named,
collapsible, statused, savable steps** — the
"render this script's run like a CI job page"
shape, which none of the parallel-process or
log-follower tools do natively.

## Upstream pointers

- Repo: <https://github.com/xavierog/moulti>
- Release notes: <https://github.com/xavierog/moulti/releases>
- License: [MIT](https://github.com/xavierog/moulti/blob/master/LICENSE) (`LICENSE`)
- Changelog: <https://github.com/xavierog/moulti/blob/master/CHANGELOG.md>
- Maintainer: [@xavierog](https://github.com/xavierog)
