# glab

- **Repo:** https://gitlab.com/gitlab-org/cli
- **Version:** v1.66.0 (2026 stable line)
- **License:** MIT ([LICENSE](https://gitlab.com/gitlab-org/cli/-/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install glab` · `scoop install glab` · `pacman -S glab` · `go install gitlab.com/gitlab-org/cli/cmd/glab@latest` · prebuilt binaries on the GitLab releases page · binary name is `glab`

## What it does

`glab` is the official **GitLab command-line client** — the GitLab
counterpart to `gh`. It wraps the GitLab REST and GraphQL APIs in
a verb/noun CLI (`glab mr create`, `glab issue list`, `glab ci view`,
`glab repo clone`) so the everyday GitLab workflow — open an MR,
review pipelines, check CI logs, comment on issues, manage labels,
trigger releases — can stay in the terminal next to `git` instead
of bouncing through the web UI. It auto-detects the current repo's
remote, knows about both `gitlab.com` and self-managed instances,
and supports multiple hosts via `glab auth login --hostname …`.

The pieces people actually use: `glab mr create -f` to open an MR
straight from the current branch with the last commit as the title;
`glab mr checkout <iid>` to fetch and switch to someone else's MR
locally; `glab ci view` for an interactive TUI of the running
pipeline (jobs, stages, live logs); `glab ci trace` to stream a
single job's log; `glab issue create -t … -d …` to file an issue
without leaving the editor; and `glab api …` for arbitrary REST
calls when no first-class verb exists yet. Auth tokens live in
`~/.config/glab-cli/config.yml` and can be scoped per host.

## Example

```bash
# from a feature branch, open an MR against the default branch
glab mr create --fill --target-branch main

# watch the resulting pipeline in a live TUI
glab ci view
```

## When to pick it / when not to

Pick `glab` if your day job is on GitLab — it is the lowest-friction
way to script MR/CI/issue flows and the only sanctioned client that
tracks the GitLab API surface release-for-release. If your repos
are on GitHub, use `gh` instead; if you want a unified client across
forges, look at `git-spice` or forge-agnostic TUIs. `glab` does not
try to abstract over GitHub vs GitLab — that is a feature, not a
bug, since it lets it expose GitLab-specific concepts (epics,
iterations, merge trains, deployment freezes) directly.
