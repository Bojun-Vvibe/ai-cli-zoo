# cmdh

> Snapshot date: 2026-04. Upstream: <https://github.com/pgibler/cmdh>

A natural-language → shell-command CLI with a small but unusual
twist: it distinguishes **interactive** vs **non-interactive**
commands, and bundles **setup commands** separately from the **target
command**. The result is a hotkey menu where you can run the setup
prerequisites alone, just the target command alone, or all of them in
sequence.

It earns a catalog slot for being the only entry that explicitly
models *"this command needs `apt install foo` first"* as a structured
output rather than burying it in prose for the human to parse.

## 1. Install footprint

- Clone + install:
  `git clone https://github.com/pgibler/cmdh.git && cd cmdh && ./install.sh`.
- TypeScript / Node — needs `node`, `npm`, `tsc` on the host.
  Installer compiles the binary and adds a `cmdh` shim to your
  `PATH` (typically via `~/.bashrc` / `~/.zshrc`).
- No PyPI / npm / Homebrew distribution; install path is git-only.
- Configuration via interactive wizard: `cmdh configure`. Writes to
  `~/.config/cmdh/config.json`. `cmdh configure show` dumps the
  current config.

## 2. Repo, version, license

- Repo: <https://github.com/pgibler/cmdh>
- Version: **no tagged releases** as of 2026-04; track the `main`
  branch HEAD. The project advertises itself as alpha-grade.
- License: MIT. License file at the repo root:
  [`LICENSE`](https://github.com/pgibler/cmdh/blob/main/LICENSE).
- Language: TypeScript (Node).

## 3. What it actually does

Invocation: `cmdh '<intent in plain English>'`. The tool sends the
intent + your detected OS (read from `/etc/os-release` or
equivalent) to the configured LLM and asks for a structured response
shape:

```jsonc
{
  "setup_commands":  ["sudo apt-get install -y jq", ...],
  "desired_command": "cat data.json | jq '.users[].email'",
  "is_interactive":  false,
  "explanation":     "Pipes the file through jq to extract emails."
}
```

The TUI then surfaces a hotkey menu:

- `s` — run **setup commands only** (the prerequisites).
- `d` — run **desired command only** (assumes prereqs satisfied).
- `a` — run **all** (setup commands then desired command).
- `c` — copy the desired command to clipboard.
- `q` — quit without running.

The `is_interactive` flag controls execution shape: non-interactive
commands are run with output captured back into the TUI;
interactive commands (e.g. `vim`, `top`, `ssh`) are spawned with
your terminal's stdio so they take over the screen normally.

## 4. MCP support

None. Does not consume MCP servers and does not expose one. The
tool's only "tool" is shell execution.

## 5. Sub-agent model

None. Single LLM call per invocation, returning the structured
JSON-shaped response in one shot.

## 6. Telemetry stance

**Off — no analytics in the binary.** The configured backend
(OpenAI, Ollama, or text-generation-webui) sees the natural-language
prompt + your OS string by construction. No update pings, no usage
counters.

## 7. Token / context strategy

- One-shot per invocation; no session, no history. Each `cmdh "…"`
  call is a fresh completion.
- The system prompt is small (<500 tokens) and constant across
  calls — it just declares the JSON output schema and the OS
  context. There is no retrieval, no embedding, no cache.
- Backend choice directly determines cost: OpenAI bills per call,
  Ollama / text-generation-webui are local-only.

## 8. Hot keybinds

| Key | Action |
|-----|--------|
| `s` | Run setup commands only |
| `d` | Run desired command only |
| `a` | Run all (setup, then desired) |
| `c` | Copy desired command to clipboard |
| `q` | Quit |

There is no in-session multi-turn revise loop; if the suggestion is
wrong you re-run `cmdh` with a clarified prompt.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Setup-vs-desired split.** Most other shell-
command CLIs ([`shell-gpt`](../shell-gpt/), [`tlm`](../tlm/),
[`gorilla-cli`](../gorilla-cli/), [`yai`](../yai/),
[`ai-shell`](../ai-shell/)) emit one line that may secretly require
three uninstalled packages. `cmdh` makes the prereqs *first-class
output* you can choose to run separately, audit, or skip. The
interactive-vs-non-interactive flag is a smaller but real win — you
don't get a `vim`-shaped command shoved into a captured-output pipe.

**Weakness.** **Maintenance velocity is low and packaging is
rough.** No tagged releases, no `npm` / Homebrew distribution,
git-clone-and-`./install.sh` only — fine for a personal box, awkward
to recommend at team scale. Provider scope is narrow (OpenAI,
Ollama, text-generation-webui — no Anthropic, no Gemini, no generic
OpenAI-compatible endpoint outside the wizard's hardcoded list).
The structured-JSON contract also means a model that fails to
produce valid JSON degrades the whole UX, with no graceful fallback
to plain text.

**Choose it when.**

- Your real complaint about other shell-command tools is "the
  suggestion *implied* I had `jq` / `k9s` / `yq` installed and I
  didn't" — `cmdh`'s setup-commands surface fixes exactly that.
- You're on a fresh dev box / VM where missing prereqs are the
  default, not the exception.
- You want to keep a `cmdh` invocation entirely local against
  Ollama with `codellama` (the documented Ollama default).

**Skip it when.**

- You want `[E]xecute / [R]evise` revise loops on the suggestion
  itself — use [`ai-shell`](../ai-shell/) (E/R/C) or
  [`yai`](../yai/) (Tab into chat).
- You need providers other than the three hardcoded backends — use
  [`mods`](../mods/) (10+ providers via Unix-pipe shape) or
  [`shell-gpt`](../shell-gpt/) (any OpenAI-compatible).
- You want a maintained, packaged tool — `cmdh` is alpha-grade, no
  releases, single-author. Treat it as a reference for the
  setup/desired pattern more than a daily driver.

## 10. Compared to neighbors in the catalog

| Tool | Setup-cmd separation | Interactive-cmd flag | Provider scope | Distribution |
|------|----------------------|----------------------|----------------|--------------|
| cmdh | Yes (`s` / `d` / `a`) | Yes | OpenAI, Ollama, text-generation-webui | git clone + `./install.sh` |
| [shell-gpt](../shell-gpt/) | No | No | Any OpenAI-compatible | `pipx` |
| [tlm](../tlm/) | No | No | Ollama-only | Single Go binary |
| [gorilla-cli](../gorilla-cli/) | No (multiple candidates instead) | No | Hosted Gorilla / OpenAI-compatible | `pipx` |
| [ai-shell](../ai-shell/) | No (single line + revise loop) | No | Any OpenAI-compatible | `npm -g` |
| [yai](../yai/) | No (Tab into chat for follow-ups) | No | OpenAI (env-override) | Single Go binary |

Decision shortcut:

- "I keep getting commands that assume packages I don't have
  installed" → `cmdh`.
- "I want to revise the suggestion before running" → `ai-shell` /
  `yai`.
- "I want N candidates to pick from" → `gorilla-cli`.
- "I want a one-binary, no-deps install" → `tlm` (Ollama) /
  `yai` (OpenAI).
