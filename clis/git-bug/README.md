# git-bug

> **Distributed bug tracker embedded in git itself — issues
> live as git objects, sync over `git push` / `git pull`, no
> server required.** Bugs, comments, labels, and identities are
> stored under a dedicated `refs/bugs/*` namespace so they
> travel with the repo, work fully offline, and survive any
> forge migration. Optional bridges import / export to GitHub,
> GitLab, Jira, and Launchpad. Pinned to **v0.10.1**
> ([LICENSE](https://github.com/git-bug/git-bug/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/git-bug/git-bug>

## TL;DR

`git bug add` opens `$EDITOR` for a new issue; the bug is
written as a chain of immutable git objects under `refs/bugs/`,
mergeable across forks like any other ref. `git bug` (no args)
launches a `bubbletea` TUI for browse / comment / label / close;
`git bug webui` boots a localhost HTML interface; `git bug
termui` and `git bug user` round out the surface. Bridges
(`git bug bridge configure --target=github`) one-way or
bidirectionally sync against external trackers, so a private
mirror can keep canonical state while a public GitHub project
remains the user-facing inbox — or you can drop the forge
entirely and ship issues with the code.

## Install

```bash
# Homebrew
brew install git-bug

# Go install (HEAD)
go install github.com/git-bug/git-bug@latest

# Pre-built binary (pin to v0.10.1)
curl -L -o /usr/local/bin/git-bug \
  https://github.com/git-bug/git-bug/releases/download/v0.10.1/git-bug_darwin_arm64
chmod +x /usr/local/bin/git-bug
```

## Example

```bash
# Create a bug, get a deterministic id (first 7 chars usually shown)
git bug add -t "panic on empty config" -m "stack trace attached"

# Browse / triage in the TUI
git bug

# Push the bug refs upstream alongside code
git bug push origin

# Pull bug refs from a teammate
git bug pull origin

# Open a localhost web UI for less-CLI-comfortable reviewers
git bug webui --port 3333

# One-time GitHub import + ongoing bidirectional sync
git bug bridge configure
git bug bridge pull
```

## When to use

- The repo must remain self-contained — air-gapped fork, public
  mirror that should not depend on a specific forge, archival
  snapshot that captures issues alongside code.
- You want offline issue triage on a plane / train without
  losing work, then sync via plain `git push`.
- A small team wants distributed bug tracking with no server
  to operate, while still bridging to GitHub / GitLab for
  external contributors.

## When NOT to use

- The team already lives in GitHub / Jira / Linear and will
  never look at a CLI — bridges work, but the second surface is
  friction without payoff.
- You need rich workflow automation (status transitions,
  approvals, integrations with CI dashboards) — that is forge
  territory.
- The repo cannot tolerate extra refs in `refs/bugs/*`
  (some hosting policies reject non-standard namespaces).
