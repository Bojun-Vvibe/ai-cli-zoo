# open-interface

> Snapshot date: 2026-04. Upstream: <https://github.com/AmberSahdev/Open-Interface>

A cross-platform desktop app that lets an LLM **drive your real
keyboard and mouse**. You type a goal in plain English ("solve
today's Wordle", "rename every screenshot in Downloads to
`YYYY-MM-DD-N.png`", "open Slack and reply to the unread DM"),
Open Interface screenshots the screen, asks a vision-capable LLM
for the next mouse/keyboard step, executes it via `pyautogui`, and
loops — re-screenshotting and course-correcting until the model
declares the task done.

It is the smallest "general-purpose computer-use agent" entry in
the catalog: one Python desktop app, one screenshot loop, one
pyautogui executor, no browser-only restriction, no Docker
sandbox.

## 1. Install footprint

- macOS: signed `.dmg` from the GitHub Releases page.
- Linux: AppImage from Releases.
- Windows: `.exe` installer from Releases.
- From source: `git clone && pip install -r requirements.txt &&
  python src/app.py` (Python 3.10+; `pyautogui` needs the OS
  accessibility / X11 permissions).
- Brings up a small floating window: API-key field, model dropdown,
  goal textbox, big "Submit" button. No CLI flags are the primary
  surface — this entry is here as the *reference computer-use
  desktop app*, not a TUI.
- API keys live in the app's settings panel:
  - OpenAI (`gpt-4o`, `gpt-4-turbo`, `gpt-4o-mini`) — default.
  - Google Gemini (`gemini-1.5-pro`, `gemini-2.0-flash`).
  - Custom OpenAI-compatible base URL (Ollama with a vision model,
    LiteLLM, OpenRouter, vLLM, LM Studio).

## 2. Repo, version, license

- Repo: <https://github.com/AmberSahdev/Open-Interface>
- Version checked: **v0.9.0** (released 2025-03-16; latest GitHub
  release as of 2026-04).
- HEAD pinned at this snapshot:
  `6a4abbe8a39f393cffa00539c6033d83476cae7d`.
- License: GPL-3.0. License file at
  [`LICENSE.md`](https://github.com/AmberSahdev/Open-Interface/blob/main/LICENSE.md).
- Not on PyPI — distributed only as platform binaries + source
  checkout.

## 3. What it actually does

The loop is intentionally tiny:

1. Take a screenshot of the current display.
2. Send `{goal, screenshot, history-of-prior-actions}` to a
   vision LLM with a prompt asking for the next discrete action
   (`click x,y`, `type "..."`, `press cmd+space`, `done`,
   `failed`).
3. Parse the action; execute it via `pyautogui` (mouse move +
   click, key press, text type).
4. Sleep briefly, screenshot again, repeat — until the model
   returns `done` or the user hits the Stop button.

There is no DOM, no accessibility tree, no element selectors. The
agent sees pixels and clicks pixel coordinates. That is both why
it works on **any application on any OS** and why it is brittle
on dense GUIs.

## 4. MCP support

None. Open Interface neither consumes MCP tools nor exposes one.
Out of scope for the project.

## 5. Sub-agent model

Single agent, single loop. No sub-agents, no planner/executor
split, no parallelism. One screenshot in, one action out, one at
a time.

## 6. Telemetry stance

Off (no analytics in the app). The configured model endpoint
sees the *full screenshot of your desktop* on every step — that
includes any open window, any visible password manager, any
secret on screen. The threat model is identical to "screen
sharing on a Zoom call to a contractor". Use a local Gemma /
Llama vision model via Ollama (`base_url=http://localhost:11434/v1`)
when the desktop is sensitive.

## 7. Token / context strategy

Each step ships a fresh full-resolution screenshot plus the
running action history as text. There is no chunking, no
summarisation, no memory store — old actions accumulate in the
prompt until the task ends or the user stops. On `gpt-4o`-class
context windows this is fine for tasks under ~30 steps; longer
tasks waste tokens replaying history.

Mitigation: split big goals ("organise Downloads then send
me a summary") into sequential smaller goals, each a fresh run.

## 8. Hot keybinds

- The floating control window is the only UI; the app registers a
  global stop hotkey (default `Ctrl+Shift+Esc`) so a runaway agent
  can be killed without grabbing the mouse back from it. **Always
  know the stop hotkey before submitting a goal.**

## 9. Killer feature, weakness, when to choose

**Killer feature.** It is the catalog's reference **OS-level
computer-use agent that targets the real desktop, not a
controlled browser**. Unlike [`browser-use`](../browser-use/) or
[`stagehand`](../stagehand/) (which drive a Playwright browser
and only see web DOMs), Open Interface drives whatever app you
have open: Finder, Slack, Photoshop, a native game. The
abstraction is "screenshot + pyautogui", which is universal.

**Weakness.** Pixel-coordinate clicks are fragile: a different
display scale, a slightly different theme, a popped-up
notification, and the model misclicks. There is no retry
heuristic beyond "screenshot again and ask again". GPL-3 desktop
binary (no headless / server mode), so it cannot be embedded in
another agent framework or driven from CI. Project release
cadence is slow (~one tagged release per year).

**When to choose.**
- You want an agent that can use **arbitrary native desktop apps**
  (not just the browser).
- You are happy with a **GUI + global hotkey** workflow, not a
  scripted/headless one.
- The desktop is non-sensitive *or* you can route the model
  through a local Ollama vision endpoint.

**When to skip.**
- Your task lives entirely in a browser → use
  [`browser-use`](../browser-use/) or
  [`stagehand`](../stagehand/) — DOM-aware, much more reliable.
- You need to embed computer-use in a larger agent / pipeline →
  use [`cua`](../cua/) (programmatic Python SDK + sandboxed VM)
  or [`self-operating-computer`](../self-operating-computer/).
- You want sandboxed execution so the agent cannot touch your real
  files → [`cua`](../cua/) (Lume / KVM VM) or
  [`ui-tars-desktop`](../ui-tars-desktop/) (in-VM agent).
- You want a TUI / scriptable interface → Open Interface has none.

## 10. Compared to neighbors in the catalog

| Tool | Surface controlled | Mechanism | Sandbox | Headless / scriptable |
|------|-------------------|-----------|---------|-----------------------|
| open-interface | Whole desktop (any app, any OS) | Screenshot + `pyautogui` | None — runs as you | No (GUI only) |
| [browser-use](../browser-use/) | Browser only | Playwright + DOM tree + screenshot | Browser process | Yes (Python) |
| [stagehand](../stagehand/) | Browser only | Playwright + DOM + LLM-suggested selectors | Browser process | Yes (TS / Python) |
| [self-operating-computer](../self-operating-computer/) | Whole desktop | Screenshot + pyautogui (CLI) | None | Yes (CLI) |
| [cua](../cua/) | Whole desktop **inside a VM** | Lume / KVM VM + agent loop | Yes (VM) | Yes (Python SDK) |
| [ui-tars-desktop](../ui-tars-desktop/) | Whole desktop (UI-TARS model) | Native vision-action model + GUI app | Optional VM | Partial |

Decision shortcut:

- "Drive my actual laptop, GUI button, fastest install" →
  `open-interface`.
- "Drive my actual laptop, scriptable from Python" →
  [`self-operating-computer`](../self-operating-computer/).
- "Drive a desktop but inside a real sandbox" → [`cua`](../cua/).
- "Drive only a browser, reliably" →
  [`browser-use`](../browser-use/) /
  [`stagehand`](../stagehand/).
