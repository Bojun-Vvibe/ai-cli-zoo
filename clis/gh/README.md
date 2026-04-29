# gh

> Snapshot date: 2026-04. Upstream: <https://github.com/cli/cli>

The official GitHub CLI: a single static Go binary that puts pull
requests, issues, releases, gists, repos, runs, projects, and the
underlying GraphQL / REST APIs on the command line as first-class
verbs. It is the catalog's reference for "talk to GitHub from a
terminal or a script", and the substrate every other AI agent CLI
in this catalog reaches for when it needs to open a PR, pull a
review comment, or trigger a workflow.

## 1. Install

- `brew install gh` — Homebrew (macOS / Linux)
- `winget install --id GitHub.cli` — Windows
- `sudo apt install gh` after adding the official apt repo
- `scoop install gh`, `choco install gh`, `dnf install gh`,
  `pacman -S github-cli`, `nix-env -iA nixpkgs.gh`
- Standalone tar.gz / zip / .deb / .rpm / .msi / .pkg per platform
  on the [releases page](https://github.com/cli/cli/releases)
- Single static Go binary, ~14 MB, no runtime dependencies. Config
  at `~/.config/gh/config.yml` and `~/.config/gh/hosts.yml`.

## 2. Version pin

**v2.92.0** (released 2026-04-28). Verify with `gh --version`.

## 3. License

MIT. SPDX: `MIT`. Full text at
[`LICENSE`](https://github.com/cli/cli/blob/trunk/LICENSE) in the
repository root.

## 4. What it does

Verbs are organised by GitHub object:

- `gh auth login | status | refresh | token` — OAuth or PAT auth
  per host; the credential helper most other tools (`act`, `gh-dash`,
  agent CLIs) silently depend on.
- `gh repo create | clone | view | fork | list | sync` — repo
  lifecycle including template-based scaffolding and bulk listing.
- `gh pr create | list | checkout | view | diff | review | merge | status`
  — the daily-driver half. `gh pr create --fill` builds a PR from
  the latest commit; `gh pr checkout 1234` puts you on the PR's
  branch with one keystroke; `gh pr merge --squash --delete-branch`
  closes the loop.
- `gh issue create | list | view | comment | close | reopen` — same
  shape for issues; `--json` + `--jq` for scripting.
- `gh release create | list | upload | download | view` — the
  standard CI release primitive (`gh release create v1.2.3 dist/*
  --notes-file CHANGELOG.md --generate-notes`).
- `gh workflow list | run | view | enable | disable` and
  `gh run list | view | watch | rerun | download | cancel` — full
  Actions control surface from the terminal.
- `gh api graphql -f query='...'` and `gh api repos/:owner/:repo/...`
  — typed access to the underlying GitHub APIs with auth and
  pagination handled, so a five-line shell script replaces a
  hand-rolled API client.
- `gh extension install | list | upgrade` — extensions surface
  community subcommands (e.g. `gh-dash`) as `gh <name>` verbs.

## 5. Install & first use

```bash
brew install gh
gh auth login                       # OAuth in the browser
gh repo clone owner/repo
cd repo
gh pr list --state open --author @me
gh pr create --fill --web           # opens the draft PR in browser
gh pr checks                        # status of CI on the current PR
gh run watch                        # live-tail the latest workflow
```

## 6. Example: scripting with `--json` + `--jq`

```bash
# every open PR I authored, sorted by oldest first
gh pr list --author @me --state open \
  --json number,title,createdAt,headRefName \
  --jq 'sort_by(.createdAt) | .[] | "\(.number)\t\(.headRefName)\t\(.title)"'

# bulk-close stale issues with a label
gh issue list --label stale --state open --json number --jq '.[].number' \
  | xargs -I{} gh issue close {} --comment "Closing as stale."

# trigger a workflow with inputs and watch it
gh workflow run release.yml -f version=v1.2.3
gh run watch
```

## 7. Why it matters in this catalog

Almost every AI coding CLI here that touches GitHub (`aider`,
`opencode`, `claude-code`, `crush`, `goose`, `pr-agent`,
`code-review-gpt`, `sweep`, `swe-agent`, `codex`, `qwen-code`,
`gemini-cli`, etc.) either shells out to `gh` for auth and
PR-creation or re-implements a thin slice of it badly. Having `gh`
on `$PATH` collapses "make a PR with the diff I just generated"
from a hand-written REST client to one subprocess call, and the
shared `~/.config/gh/hosts.yml` token store means the agent does
not need its own credential manager. For human-in-the-loop review
loops, `gh pr view --comments`, `gh pr diff`, and `gh pr review` are
the keyboard-only review surface that matches an editor-resident
agent.

## 8. Alternatives

- [`glab`](../glab/) — same shape for GitLab; pair both on a
  multi-forge workstation.
- [`gh-dash`](../gh-dash/) — Bubble Tea TUI built **on top of** `gh`
  for prioritising your PR / issue queue across many repos.
- Forgejo / Gitea CLIs (`tea`) for self-hosted forges.
- The raw GitHub REST / GraphQL API via `curl` + `jq` — what `gh
  api` saves you from writing.

## 9. Telemetry

No analytics SDK in the binary. Outbound network calls are
exclusively to the GitHub host(s) configured in `hosts.yml` and (for
extensions) the package origin you opted into. `gh config set
prompt disabled` for fully non-interactive CI use.
