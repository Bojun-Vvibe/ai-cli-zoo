# git-graph

> **Render a clear, branching-model-aware ASCII / SVG commit
> graph from any git repo** — a single Rust binary that walks
> the commit DAG and lays it out so the *first-parent main
> line* stays straight, *feature branches* fan out and merge
> back as side-tracks, and *merges* are obvious; supports
> built-in branching models (`git-flow`, `simple`, `none`) or
> a custom `git-graph.toml`, multiple output styles
> (`thin` / `round` / `bold` / `double` / `ascii`), colour
> profiles, optional commit messages, and an SVG export — all
> from one command, no TUI session required. Pinned to
> **v0.7.0** (commit
> `49ed50a1fc6aff4c4ff2da71fdbb5b3324962961`,
> [LICENSE](https://github.com/git-bahn/git-graph/blob/master/LICENSE),
> SPDX: `MIT`).

Source: <https://github.com/git-bahn/git-graph>

## TL;DR

`git-graph` renders the commit graph the way you'd draw it on
a whiteboard: `main` is a vertical line down the middle,
feature branches are horizontal off-shoots that loop back at
their merge commit, and the layout is stable across re-runs
so you can paste it into a PR review or a postmortem and have
it look the same. Out of the box it understands `git-flow`
(treats `develop` as the integration line, `release/*` and
`hotfix/*` as side-tracks, tags `master`); pass `--model
simple` for a plain "one mainline + feature branches" layout
or write a `git-graph.toml` to encode any team convention
(branch-name globs → role + colour + lane). Output is colour
ANSI by default; `--style ascii` for paste-into-Markdown,
`--svg out.svg` for an inline asset, `--no-color` for log
files. It is a one-shot CLI, not a TUI — runs to completion
and exits, ideal for `git alias`, hooks, and CI artefacts.

## Install

```bash
# Cargo
cargo install git-graph

# Arch (AUR)
yay -S git-graph

# Pre-built binary (Linux / macOS / Windows)
# https://github.com/git-bahn/git-graph/releases/tag/v0.7.0

# verify
git-graph --version    # git-graph 0.7.0
```

Optional but recommended:

```bash
# wire as a git alias
git config --global alias.gr '!git-graph --style round --model git-flow'
git gr        # now prints the branching-model-aware graph
```

## License

MIT — see
[LICENSE](https://github.com/git-bahn/git-graph/blob/master/LICENSE).
Permissive, no attribution requirement on graph output;
redistribute the binary freely.

## Representative Commands

```bash
# 1. default: detect a model, print colour graph to terminal
git-graph

# 2. force a model (git-flow, simple, none)
git-graph --model git-flow
git-graph --model simple

# 3. show commit messages alongside the graph
git-graph --format oneline
git-graph --format short

# 4. ASCII-only output for pasting into Markdown / GitHub PRs
git-graph --style ascii --no-color > graph.txt

# 5. render to SVG for a postmortem doc / wiki
git-graph --svg release-2026-04.svg --model git-flow

# 6. limit to last 50 commits on a wide repo
git-graph --max-count 50

# 7. custom model: drop git-graph.toml in the repo root with
#    branch-name regex → role mapping; git-graph picks it up
git-graph    # uses ./git-graph.toml if present
```

## Why It Matters

`git log --graph --oneline --all --decorate` is the lingua
franca for "show me the branch shape," but on any non-trivial
repo the layout collapses into a tangle: parallel branches
crisscross unpredictably, the mainline weaves left-right with
each merge, and the same commit looks different between two
runs. Tools like [`gitui`](../gitui/) and
[`lazygit`](../lazygit/) ship interactive TUIs but do not
print a stable shareable graph. `git-graph` solves the narrow
problem of *producing a clear, branching-model-aware,
copy-pasteable graph from one command*: the mainline is
always vertical, feature branches always go to one side,
merges land where you expect, and the colour / style is
configurable for the team's convention. Pairs with
[`onefetch`](../onefetch/) (per-repo summary card with
contributors / language breakdown / age, also one-shot),
[`tig`](../tig/) (curses log browser when you want
*interactive* navigation rather than a static rendering),
and [`gitui`](../gitui/) / [`lazygit`](../lazygit/) (full
TUI workflows). Reach for `git-graph` when the deliverable is
**a static graph artefact** — a PR comment showing how a
release branch landed, a `release-notes.md` section, a CI
job that drops `branch-graph.svg` next to the build log, or
a screenshot for a postmortem — and the project follows a
convention worth honouring (`git-flow`, `release/*` lanes, a
custom branching model). The killer property is **stable,
model-aware layout in a one-shot CLI**: same input → same
graph → safe to commit into docs, paste into reviews, and
regenerate from CI without surprise re-flows.
