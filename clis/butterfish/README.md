# butterfish

> Snapshot date: 2026-04. Upstream: <https://github.com/bakks/butterfish>

A Go-built **shell wrapper with AI superpowers** — `butterfish shell`
launches a transparent subshell that records every prompt and command
output, then lets you ask the model questions about what just happened
or autocomplete the next command via a hotkey. Sits in a different
slot than `aichat` / `mods` / `tlm`: those are tools you *invoke*; this
one *is* your shell session, watching everything you do, ready to be
asked "why did that fail?" without copy-pasting scrollback.

## 1. Install footprint

- `brew install butterfish` (the Homebrew formula tracks upstream),
  or `go install github.com/bakks/butterfish/cmd/butterfish@latest`,
  or a release binary from
  <https://github.com/bakks/butterfish/releases>.
- Single static Go binary, no runtime deps. Config at
  `~/.config/butterfish/butterfish.env`.
- `butterfish shell` replaces your interactive shell for the session;
  `exit` returns to the parent shell unchanged. No daemon.

## 2. Repo + version + license

- Repo: <https://github.com/bakks/butterfish>
- Latest release: **v0.3.13**
- License: **MIT** —
  <https://github.com/bakks/butterfish/blob/main/LICENSE>
- Default branch: `main`

## 3. Models supported

OpenAI (default) and any **OpenAI-compatible** endpoint via
`OPENAI_API_BASE` — Ollama, LM Studio, vLLM, LiteLLM, OpenRouter,
Groq, DeepSeek. No first-class Anthropic / Gemini SDK; you reach
those via a LiteLLM-style proxy.

## 4. MCP support

None. The "tools" the model can use are the actual Unix commands
in your shell; there is no MCP client or tool-spec layer.

## 5. Sub-agent model

None — single conversation pinned to the shell session. The model
sees a sliding window of recent commands + outputs.

## 6. Telemetry stance

Off, no opt-in. Outbound traffic is exclusively to the configured
OpenAI-compatible endpoint.

## 7. Killer feature, weakness, when to choose

**Killer feature.** `butterfish shell`'s **goal mode**: prefix a
line with `!` and the model proposes a shell command in your prompt
ready for `Enter`; prefix with `!!` and it auto-executes a multi-step
goal, feeding each command's output back as context. Combined with
"hit a key to ask about the last command's output without leaving
the shell", it eliminates the alt-tab-to-ChatGPT loop for everyday
shell debugging.

**Weakness.** Shell-coupling is fragile around exotic prompts
(starship / oh-my-zsh themes with ANSI escapes), screen / tmux
sessions inside the wrapper can confuse the scrollback capture, and
non-OpenAI providers need an env-var dance. Not an editor — for
"refactor this file", reach for `aider` / `opencode` / `claude-code`.

**When to choose.** You live in a terminal, you want
**ask-about-last-output** and **natural-language-to-command** as
keystrokes inside your normal shell, and you do not want a separate
TUI window or REPL. Pick `tlm` or `shell-genie` instead if you only
want one-shot NL→command without the wrapper. Pick `wut` instead if
you want the same "explain last output" behavior without replacing
your shell.

## 8. Quick usage

```sh
# install
brew install butterfish

# configure
echo 'OPENAI_API_KEY=sk-...' > ~/.config/butterfish/butterfish.env

# launch wrapped shell
butterfish shell

# inside the wrapped shell:
#   !find all png larger than 1MB                # NL → suggested command
#   !!compress every jpg in this dir to webp     # autonomous multi-step
#   <Ctrl-X G>                                   # ask about last output

# one-shot mode (no wrapper)
butterfish prompt 'explain the difference between rsync -a and -z'
```
