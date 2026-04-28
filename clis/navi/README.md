# navi

> **Interactive cheatsheet tool for the command-line** — a fuzzy-finder
> over your own (and the community's) annotated one-liners. Pinned to
> **v2.24.0** (commit
> `1ac218cb1e0e80649ef23c8a916e67efc3086833`,
> [LICENSE](https://github.com/denisidoro/navi/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/denisidoro/navi>

## TL;DR

`navi` is the answer to "I know there is a `tar` / `ffmpeg` / `kubectl` /
`openssl` invocation that does this, I have used it before, and I cannot
remember the flag order." You write (or `git clone`) `.cheat` files: each
entry is a `% tag` header, a one-line `# description`, the actual command
template with `<placeholder>` slots, and optional `$ placeholder: shell pipeline`
blocks that pre-populate each slot via fzf. At runtime `navi` opens an
fzf-style picker over every cheat in your search path; you select one,
fill in the placeholders interactively (each placeholder gets its own fzf
panel populated by the `$ placeholder:` pipeline — e.g. listing your AWS
profiles, your kube contexts, your ssh hosts), and the assembled command
either runs immediately or is printed for you to paste. The killer detail
is that it ships a **widget** (`navi widget zsh|bash|fish`) that binds to
`Ctrl-G` so the picker pops up *inside your current prompt* — no subshell,
no copy-paste, the rendered command lands on your readline buffer.

## Install

```bash
# Homebrew (macOS / Linux)
brew install navi

# Cargo
cargo install --locked navi

# Arch
pacman -S navi
```

Bind the widget:

```bash
# in ~/.zshrc
eval "$(navi widget zsh)"
```

Pull community cheats:

```bash
navi repo browse           # fuzzy-pick from the curated list
navi repo add denisidoro/cheats
```

## Niche

The "**spaced-repetition for shell muscle memory**" slot. Where
[`tldr`](../tealdeer/) gives you a *static, community-curated* page of
canonical examples for one tool, and where `fzf`'s `Ctrl-R` searches your
*own* shell history (which is full of half-finished, wrong, or
context-specific invocations), `navi` sits in between: a *typed*,
*placeholder-aware*, *per-team* cheat library that you author once and
then never have to re-derive. The `$ placeholder: pipeline` syntax is the
real innovation — a cheat is not a static string, it is a small program
that asks fzf to pick the right argument from your live environment, so
"`kubectl logs <pod>`" turns into a picker over `kubectl get pods -o name`
without you typing a thing.

## Why it matters

- **Fuzzy-pickable, parameterised cheats** — every placeholder gets its
  own fzf panel, populated by a shell pipeline you control. Cheats stop
  being "examples to copy" and start being "interactive forms".
- **Widget mode is real** — `Ctrl-G` invokes the picker on top of your
  current prompt and writes the assembled command to your readline
  buffer. No subshell, no `$()`, the command is yours to edit before you
  hit Enter.
- **Repo-level cheat sharing** — `navi repo add owner/repo` clones a git
  repo of `.cheat` files into `~/.local/share/navi/cheats/`; teams
  publish their internal runbook one-liners as a regular GitHub repo and
  every dev gets them with one command.
- **Apache-2.0, single Rust binary** — no daemon, no index, no telemetry;
  startup is dominated by fzf, not navi.
