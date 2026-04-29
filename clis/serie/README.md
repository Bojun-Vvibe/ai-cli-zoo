# serie

> **A rich `git log --graph` reimplemented as a fast TUI** —
> renders the full commit graph of a repository in the
> terminal with color-coded branches, ref decorations, and
> an inspectable side panel for the selected commit's
> message, diff, and changed files. Pinned to **v0.7.2**
> ([LICENSE](https://github.com/lusingander/serie/blob/main/LICENSE),
> MIT).

Source: <https://github.com/lusingander/serie>

## TL;DR

`serie` is what you reach for when `git log --graph
--all --oneline --decorate` becomes unreadable on a
real-world repo with dozens of long-lived branches.
It's a single Rust binary written against `gitoxide` /
`git2` that parses the full commit DAG once at startup
and then renders it as an interactive `ratatui` view:
the left pane is the colored ASCII graph (one row per
commit, branches as continuous vertical lines, merges
as joins), and the right pane is a tabbed inspector
showing the commit message, the unified diff, the list
of changed files, and the refs (branches, tags, remotes)
that point at the selected commit. Navigation is `j/k`
or arrow keys; `/` opens an incremental search across
commit messages and authors; `Enter` on a file in the
diff pane drills into a per-file view; `y` yanks the
selected commit's hash to the system clipboard. Unlike
`tig`, the rendering is GPU-friendly (no curses
flicker), unicode-correct (works with CJK author names
and commit messages), and the graph layout algorithm is
deterministic — re-running on the same repo produces a
byte-identical screenshot, which makes it usable in
documentation. The whole thing is ~5 MB statically
linked, no runtime deps beyond `git` itself, and starts
in under 200 ms even on a repo with 100k+ commits
because the graph is rendered lazily as you scroll.

## Install

```bash
# Homebrew (macOS / Linux)
brew install serie

# Cargo
cargo install --locked serie

# Nix
nix-env -iA nixpkgs.serie

# Arch (AUR)
yay -S serie

# from a release binary
curl -LO https://github.com/lusingander/serie/releases/download/v0.7.2/serie-v0.7.2-x86_64-apple-darwin.tar.gz
tar xzf serie-v0.7.2-x86_64-apple-darwin.tar.gz
sudo install serie /usr/local/bin/

# verify
serie --version    # serie 0.7.2
```

## Basic usage

```bash
# open in current repo
cd ~/code/some-repo
serie

# open a specific repo
serie -p ~/code/other-repo

# pick a graph layout style (default | symbols)
serie --graph-width 32

# preset for monochrome terminals
serie --color-set monotone

# inside the TUI:
#   j / k or ↓ / ↑   move between commits
#   /                 incremental search across messages + authors
#   Tab / Shift-Tab   cycle right-pane tabs (message / diff / files / refs)
#   Enter             drill into the selected file's diff
#   y                 yank the selected commit hash to clipboard
#   ?                 keybinding help overlay
#   q                 quit
```

The layout, color set, and keybindings are all configurable
via `~/.config/serie/config.toml` — see the upstream
`README.md` for the full schema.

## When to choose

- **Your repo has too many branches for `git log --graph`
  to be useful** — the moment you have 8+ active branches
  and a few merge commits per day, the ASCII graph from
  upstream `git` becomes a wall of `|/\\`. `serie`'s
  rendering algorithm assigns each branch a stable
  column and color, so the same branch stays in the
  same lane as you scroll.
- **You want to inspect commits without leaving the
  graph** — the right-pane tabs (message / diff / files
  / refs) mean you can review a series of commits the
  way you would in a GitHub PR diff view, but locally
  and without spinning up a web UI.
- **You're recording a screencast or writing
  documentation** — the deterministic rendering and
  clean color palette make `serie` screenshots
  significantly more legible than `tig` or
  `git log --graph` output, and the `--color-set
  monotone` flag gives you a print-friendly version
  for PDFs and slide decks.
- **You're on a slow / remote terminal** — `serie`
  renders one screen at a time and pages the rest, so
  even on a high-latency SSH session it stays
  responsive on repos that would freeze a `tig`
  session for several seconds.

## When NOT to choose

- **You want to *write* commits, not just read them**
  — `serie` is read-only by design. For staging, committing,
  rebasing, and branch ops, reach for
  [`gitui`](../gitui/), [`lazygit`](../lazygit/), or
  [`gitu`](../gitu/), all of which include a graph view
  alongside full porcelain.
- **You need cross-repo or PR-aware browsing** — `serie`
  knows about one repo on disk and nothing about
  GitHub / GitLab / forge state. Use
  [`gh-dash`](../gh-dash/) for cross-repo PR/issue
  triage or [`glab`](../glab/) for GitLab.
- **You only ever look at the most recent few commits**
  — `git log -n 20 --oneline` in a regular shell is
  faster to invoke and pipe into `grep` / `fzf`. `serie`
  earns its keep on history-spelunking sessions, not on
  one-line "what did I just commit" checks.
- **You need `jj` / `sapling` / non-`git` VCS support**
  — `serie` parses git's object store directly and has
  no notion of jj operations or sapling commit graphs.
  Use [`lazyjj`](../lazyjj/) for `jj`.

## Why it fits the zoo

The zoo collects developer-tooling CLIs that *replace
something an average engineer types every day with a
denser, faster, more visual version*. `git log --graph
--all --oneline --decorate` is exactly that kind of
incantation: a long, hard-to-remember default that
produces output most people scroll past. `serie` takes
the same data and renders it as something you can
actually navigate, while staying a single static binary
with no runtime dependencies and no network surface.
That makes it a clean companion to existing zoo entries
like [`gitui`](../gitui/), [`gitu`](../gitu/),
[`lazygit`](../lazygit/), [`tig`](../tig/),
[`git-branchless`](../git-branchless/), and
[`onefetch`](../onefetch/) — different facets of "make
git history legible without leaving the terminal."

## Upstream pointers

- Repo: <https://github.com/lusingander/serie>
- Release notes: <https://github.com/lusingander/serie/releases>
- License: [MIT](https://github.com/lusingander/serie/blob/main/LICENSE)
- Config schema: <https://github.com/lusingander/serie#configuration>
- Author: [@lusingander](https://github.com/lusingander)
  (also wrote [`stu`](https://github.com/lusingander/stu),
  the S3 TUI).
