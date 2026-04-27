# gh-dash

> **Customisable terminal dashboard for `gh`** — a Bubble Tea TUI
> that turns the GitHub CLI into a multi-section pull-request and
> issue inbox. Declare any number of search-query-backed sections
> in `~/.config/gh-dash/config.yml` (e.g. "PRs awaiting my review,"
> "my open PRs," "issues I'm assigned in repo X," "PRs failing
> CI") and `gh dash` opens them as keyboard-navigable tables with
> live status, CI checks, review state, and one-keystroke actions
> (checkout, comment, approve, merge, close, open-in-browser).
> Pinned to **v4.23.2** (commit
> `9ac6fc99b74d8788f6c371d87fda1485f01ba590`,
> [LICENSE.txt](https://github.com/dlvhdr/gh-dash/blob/main/LICENSE.txt),
> MIT).

Source: <https://github.com/dlvhdr/gh-dash>

## TL;DR

`gh-dash` is what you reach for when "open ten browser tabs to
triage today's PRs" stops scaling. It runs as a `gh` extension
(`gh extension install dlvhdr/gh-dash` then `gh dash`), reuses
your existing `gh auth` token, and opens a TUI with a configurable
set of *sections* — each section is a saved GitHub search query
that becomes a sortable table of PRs or issues. Default sections
ship "My pull requests," "Needs my review," and "Subscribed," but
the value is in the YAML: anything `gh search prs` /
`gh search issues` accepts becomes a section, so you can build
"PRs failing CI in my org," "issues with `priority:high` and no
assignee in repo X," "PRs older than 14 days waiting on me."
Inside a section, arrow keys move row-to-row, `Enter` opens a
detail pane with diff + comments + checks, `c` checks out the PR
locally (`gh pr checkout`), `o` opens it in a browser, `a`
approves, `m` merges, `/` filters live. Layout, colors, and
keybinds are themable in the same YAML.

## Install

```bash
# as a gh extension (the supported install)
gh extension install dlvhdr/gh-dash

# Homebrew (installs both gh and gh-dash, then wire up)
brew install dlvhdr/formulas/gh-dash

# from a release binary (any OS, no gh dependency for the binary itself,
# but you still need `gh auth login` for API access)
curl -Lo gh-dash.tar.gz "https://github.com/dlvhdr/gh-dash/releases/download/v4.23.2/gh-dash_4.23.2_darwin_arm64.tar.gz"
tar xf gh-dash.tar.gz gh-dash
sudo install gh-dash /usr/local/bin/

# verify (as gh extension)
gh dash --version    # gh-dash version 4.23.2

# verify (as standalone)
gh-dash --version
```

First boot reads `~/.config/gh-dash/config.yml`; if it does not
exist, `gh-dash` generates a sensible default with three sections
and opens. Edit the YAML to add / remove / reorder sections.

## License

MIT — see
[LICENSE.txt](https://github.com/dlvhdr/gh-dash/blob/main/LICENSE.txt).
Permissive, no attribution required for binaries; redistribute
freely.

## One Concrete Example

```yaml
# ~/.config/gh-dash/config.yml
prSections:
  - title: Needs my review
    filters: is:open review-requested:@me archived:false
    limit: 50
  - title: My open PRs
    filters: is:open author:@me archived:false
  - title: PRs failing CI in my org
    filters: is:open org:my-org status:failure archived:false
    limit: 30
  - title: Stale PRs (>14d)
    filters: is:open updated:<2025-12-01 author:@me archived:false

issuesSections:
  - title: Assigned to me
    filters: is:open assignee:@me archived:false
  - title: My team's bug backlog
    filters: is:open label:bug team:my-org/my-team archived:false

defaults:
  preview:
    open: true
    width: 60
  prsLimit: 20
  view: prs

keybindings:
  prs:
    - key: M
      command: >
        gh pr merge {{.PrNumber}} --squash --auto --repo {{.RepoName}}
    - key: D
      command: >
        gh pr checkout {{.PrNumber}} --repo {{.RepoName}}
        && code .
```

```bash
gh dash                # opens with the YAML above
# inside:
#   tab / shift-tab  cycle sections
#   j / k or arrows  row navigation
#   enter            open detail pane (diff + checks + comments)
#   /                filter the current section live
#   c                checkout PR locally (gh pr checkout)
#   a                approve
#   o                open in browser
#   r                refresh current section
#   R                refresh all sections
#   ?                help overlay
#   q                quit
```

## Niche It Fills

**A keyboard-driven, query-driven inbox for GitHub work that
lives in the same terminal as your editor.** The web UI's
"Pulls" / "Issues" pages are tab-and-mouse-heavy, do not let
you persist named saved-search views beyond your own profile
filter, and force a context switch out of the terminal every time.
`gh-dash` collapses the whole "what do I owe people on GitHub
today" loop into a single TUI you can pin in a `tmux` /
[`zellij`](../zellij/) pane, with sections that *are* saved
search queries declared as YAML you check into your dotfiles. For
an operator who reviews 5-20 PRs a day across multiple repos and
orgs, this is the difference between "a 30-minute browser-tab
ritual every morning" and "open the dashboard, drive with the
keyboard, done."

## Why use it

Three things `gh-dash` does that `gh pr list` / the web UI do not,
that pay back the switching cost:

1. **Multiple parallel saved-search sections in one view.** `gh pr
   list` shows one repo's open PRs; the web UI's saved search is
   one filter at a time. `gh-dash` shows N sections side-by-side
   and `Tab`-cycles between them, so "PRs awaiting my review"
   plus "my open PRs" plus "stale PRs" are three keystrokes apart
   without re-querying. The YAML config lives in your dotfiles —
   so the same dashboard moves with you to a new laptop.
2. **One-keystroke actions on the row under the cursor.**
   Approving, merging, checking out, commenting, opening in
   browser are all single keys (configurable in YAML), and the
   `command:` field in `keybindings:` lets you bind any shell
   command with `{{.PrNumber}}` / `{{.RepoName}}` /
   `{{.HeadRefName}}` template variables — e.g. "checkout the PR
   *and* open VS Code in the worktree" as one key.
3. **Live CI status + review state per row.** Each row shows
   check status (green / red / pending), review status (approved
   / requested-changes / awaiting), and lines added / removed at
   a glance. The web UI hides this behind a click into each PR;
   `gh dash` shows the whole grid at once, so "which PR is
   blocked on what" is visible without drilling in.

For an LLM-CLI workflow where you have an agent opening PRs from
worktrees and you need to triage which ones merged cleanly vs
which need human attention, `gh dash` with a "PRs from automation
account" section is the cheapest review queue you can build.

## Vs Already Cataloged

- **Vs [`lazygit`](../lazygit/):** `lazygit` is a TUI for the
  *local* git repo (commits, branches, stashes, rebases);
  `gh-dash` is a TUI for the *remote* GitHub state (PRs, issues,
  reviews, CI). They compose: `lazygit` for "shape this branch
  before pushing," `gh-dash` for "triage what's already on the
  hub." `gh dash` even shells out to `gh pr checkout` so the PR
  you pick lands as a local branch `lazygit` then operates on.
- **Vs [`jj`](../jj/):** `jj` is a Mercurial-style VCS frontend
  (operating on local commits / changes / conflicts); `gh-dash`
  is a code-review-queue TUI. Different layers, no overlap.
- **Vs [`pr-agent`](../pr-agent/):** orthogonal — `pr-agent` is
  an LLM-driven PR *reviewer* that posts AI comments back to the
  PR; `gh-dash` is a *human triage* surface for the resulting PR
  list. A common workflow: `pr-agent` runs in CI on every PR and
  drops a `/review` summary, then you open `gh dash` to walk the
  queue and decide which AI suggestions to act on.
- **Vs [`code-review-gpt`](../code-review-gpt/):** same
  orthogonality as above — `code-review-gpt` produces inline AI
  review comments at PR time, `gh-dash` is the keyboard-first
  surface where humans triage the resulting threads.
- **Vs `gh pr list` / `gh search` (built-in `gh`):** `gh pr list`
  is one query, one repo, one render-and-exit; `gh-dash` is N
  persistent queries with live refresh, a detail pane, and
  keyboard actions. Pick raw `gh` for scripting (`gh pr list
  --json …`); pick `gh dash` for the day-to-day inbox.

## Caveats

- **GitHub-only.** No GitLab / Bitbucket / Gitea / Forgejo
  backends — `gh-dash` is built on `gh`'s GraphQL surface.
  Cross-SCM workflows need a different tool (or a per-SCM TUI
  per pane).
- **Section queries cost API calls.** Each section you declare is
  a separate search query refreshed on `R` (and on a configurable
  interval). Ten sections × auto-refresh × secondary GraphQL
  fetches for CI / reviewer state can hit the GitHub primary /
  search rate limits on a busy dashboard. Keep total sections
  ≤ ~10 and tune `defaults.refetchIntervalMinutes`.
- **The detail pane is read-mostly.** You can comment, approve,
  merge — but writing a multi-paragraph review with suggestions
  is still nicer in the browser or via `gh pr review --body-file`.
  `gh-dash` shines at triage; the web UI still wins at long-form
  review authoring.
- **Custom keybindings run shell commands as the current user.**
  `command:` template strings interpolate PR / repo fields and
  shell-execute — treat the YAML the same way you treat a shell
  rc file (do not source it from untrusted sources). The provided
  templates are well-escaped but a hand-rolled `command:` with
  unquoted `{{.HeadRefName}}` is a shell injection foothold if
  someone names a branch creatively.
- **No write access without `gh auth` scopes.** `a` / `m` / `c`
  use `gh`'s token; if your token lacks `repo` / `write:org`,
  the action silently fails or errors at the `gh` layer. Run
  `gh auth status` and re-run `gh auth refresh -s repo` when
  actions stop working.
- **Section limits are per-section.** A `limit: 50` section will
  only show 50 rows even if 200 match; there is no in-TUI "load
  more." Raise the limit if you need a longer view (at the cost
  of API quota).
