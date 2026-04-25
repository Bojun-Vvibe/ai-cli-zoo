# tabby

> Snapshot date: 2026-04. Upstream: <https://github.com/TabbyML/tabby>
> Binary name: `tabby`

`tabby` is a **self-hosted AI coding assistant server** — think
"hosted-inline-completion-shaped backend that you run on your own
GPU" rather than a CLI you talk to like
[`aider`](../aider/) or [`opencode`](../opencode/). The server speaks
the OpenAI-style completion / chat protocol plus a Tabby-specific
inline-completion contract, with first-party IDE plugins (VS Code,
JetBrains, Vim/Neovim, Emacs) connecting back to it. The CLI surface
exists for **operating the server**: `tabby serve --model
StarCoder-1B --device cuda`, `tabby download`, `tabby scheduler` for
the repo-indexing job that powers retrieval-augmented completion.

What earns it a slot in this catalog: `tabby` is the **server-side
counterpart** to most other entries here. Everything else is "client
agent → call somebody else's API"; `tabby` is "stand up the API
yourself, on prem, pointed at open-weights coder models, with
repo-aware retrieval indexed by a built-in scheduler". You then plug
[`continue`](../continue/), [`aider`](../aider/) (`--openai-api-base
http://tabby:8080/v1`), or your IDE plugin into it.

## 1. License

Source-available — root `LICENSE` is split: most of the codebase is
Apache-2.0, while `ee/` directories are under the Tabby Enterprise
Edition license (paid features: SSO, multi-tenant). `gh repo view`
reports the SPDX as "Other" because of the dual layout. For a
single-tenant self-host on the OSS surface, treat it as Apache-2.0.

## 2. Latest version

`v0.32.0` (released 2026-01-25).

## 3. Install

```bash
# Docker (recommended for GPU hosts):
docker run -it --gpus all -p 8080:8080 -v $HOME/.tabby:/data \
  tabbyml/tabby serve --model StarCoder-1B --device cuda

# macOS / Linux native:
brew install tabbyml/tabby/tabby
tabby serve --model StarCoder-1B --device metal   # or --device cpu

# Then point your IDE plugin / OpenAI-compatible client at:
# http://localhost:8080
```

Repo-aware completions: `tabby scheduler --now` to index a connected
git repository (configured via the admin UI at `/`) so completions can
cite cross-file context.

## 4. When to choose

- You want **on-prem inline-completion + chat** for a team, with the
  model weights and the prompt stream never leaving your network.
- You already operate GPU hosts and would rather **own the inference
  endpoint** than route every keystroke through a third-party API.
- You want to **reuse it as a generic OpenAI-compatible backend** for
  [`aider`](../aider/), [`continue`](../continue/),
  [`shell-gpt`](../shell-gpt/), etc. via `--openai-api-base`.
- Avoid if you just want a single-developer terminal coding agent
  ([`opencode`](../opencode/), [`crush`](../crush/),
  [`aider`](../aider/) are right-sized) or if you are not prepared to
  operate a long-running indexing daemon.
