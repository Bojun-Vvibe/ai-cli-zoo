# ghq

> **Remote-repository organiser — clone every repo into a
> predictable `~/ghq/<host>/<owner>/<repo>` tree, then jump to any
> of them by partial name** — a Go-implemented CLI that turns
> "where did I clone that repo last week?" into a non-question by
> imposing one root directory (`ghq.root`, default `~/ghq`) under
> which every clone lives at a deterministic path derived from the
> remote URL: `ghq get https://github.com/foo/bar` becomes
> `~/ghq/github.com/foo/bar`, `ghq get gitlab.example.com/team/svc`
> becomes `~/ghq/gitlab.example.com/team/svc`. Composes with `fzf`
> via the canonical `ghq list | fzf` one-liner so `Ctrl+G` in your
> shell becomes "fuzzy-pick any of my 400 cloned repos and `cd`
> into it." Pinned to **v1.10.1** (released 2026-04-11,
> [LICENSE](https://github.com/x-motemen/ghq/blob/master/LICENSE),
> MIT).

Source: <https://github.com/x-motemen/ghq>

## TL;DR

`ghq` is what you reach for when `~/code/`, `~/src/`, `~/Projects/`,
`~/dev/`, and `~/Desktop/oneoff/` all contain git repos cloned at
different times under different conventions, and the answer to
"do I already have that repo cloned?" requires `find ~ -name .git
-type d 2>/dev/null` and several seconds of staring. `ghq get
<url>` clones into the deterministic location; `ghq list` prints
every cloned repo as one line per repo (relative to `ghq.root`);
`ghq list -p` prints absolute paths; `ghq look <pattern>` `cd`s
into the matching repo (must `eval` the shell integration).
Mirrors the entire history of one GitHub user / org with
`ghq get --look-mirror github.com/foo` and the v1.10.1 update
makes `ghq rm` worktree-aware so removing a colocated `jj` /
git-worktree repo no longer leaves orphan dirs.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ghq

# Direct binary download (macOS arm64 example)
curl -LO https://github.com/x-motemen/ghq/releases/download/v1.10.1/ghq_darwin_arm64.zip
unzip ghq_darwin_arm64.zip && sudo install ghq /usr/local/bin/

# Go install
go install github.com/x-motemen/ghq@v1.10.1

# Arch
pacman -S ghq
```

## Usage

```bash
# Configure the root once (or rely on the ~/ghq default)
git config --global ghq.root ~/src

# Clone deterministically — ends up at ~/src/github.com/dbcli/pgcli
ghq get https://github.com/dbcli/pgcli

# Clone with a custom remote name (sshalias) — same destination path
ghq get git@github.com:dbcli/pgcli.git

# List every repo (relative to ghq.root) for piping into fzf
ghq list

# The canonical fzf integration — bind to Ctrl+G in ~/.zshrc
fzf-ghq() {
    local repo
    repo=$(ghq list | fzf --preview "ls $(ghq root)/{}") || return
    cd "$(ghq root)/$repo"
}
zle -N fzf-ghq && bindkey '^G' fzf-ghq

# Bulk-clone every public repo of an owner (useful for org migrations)
ghq get --look-mirror github.com/dbcli

# Remove a repo and its empty parent directories (worktree-aware in v1.10.1)
ghq rm github.com/foo/bar
```

## Why pick this

The closest peers are [`gh repo clone`](https://cli.github.com/)
(GitHub-only, dumps into `pwd`, no organising convention) and
shell aliases over `git clone` (no jump, no list). `ghq` is the
*opinionated layout* — once every machine you touch has the same
`~/ghq/<host>/<owner>/<repo>` tree, "where is that repo?" is no
longer a question on any of them, and the `ghq list | fzf` jump is
the single most-pressed shortcut for many of the people who use
it. For an LLM-CLI workflow the deterministic path means a
`CLAUDE.md` / `AGENTS.md` can reference `~/ghq/github.com/foo/bar`
literally without "well, depends where the user cloned it";
agents that need to spider across a known set of related repos
(internal monorepo + N satellite microservice repos) get a
predictable filesystem layout to walk. Pairs naturally with
[`zoxide`](../zoxide/) (zoxide ranks frequently-visited dirs;
ghq guarantees the dir exists at a predictable path in the first
place).
