# act

- **Repo:** https://github.com/nektos/act
- **Version:** v0.2.87 (2026-04-01)
- **License:** MIT ([LICENSE](https://github.com/nektos/act/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install act` · `gh extension install nektos/gh-act` · `curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash` · prebuilt binaries on the GitHub release page (`act_Darwin_arm64.tar.gz`, `act_Linux_x86_64.tar.gz`) · binary name is `act`

## What it does

`act` runs your GitHub Actions workflows locally inside Docker
containers, reading the same `.github/workflows/*.yml` files the
hosted runners use, so the inner-loop iteration on a CI failure
shrinks from minutes-per-push to seconds-per-rerun.

- Discovers workflows in `.github/workflows/` and resolves the event
  payload (`act push`, `act pull_request`, `act workflow_dispatch
  -W .github/workflows/release.yml`). Custom event JSON via
  `-e event.json` for things like `release` or `issue_comment`.
- Pulls runner images (`catthehacker/ubuntu:act-latest` by default;
  `-P ubuntu-latest=...` to override) and executes each job inside a
  fresh container. Matrix builds, services, and reusable workflows
  (`uses: ./.github/workflows/foo.yml`) all work locally.
- `act -l` lists detected jobs without running anything; `-j <job>`
  scopes execution to a single job; `-s GITHUB_TOKEN=…` injects
  secrets without writing them to disk; `--bind` mounts the working
  copy directly so you iterate on action code without commits.
- Caches actions in `~/.cache/act` so subsequent runs only pull
  net-new dependencies. `act --reuse` keeps the runner container
  alive across runs for snappy re-execution.
- `act-cli` ships installers for Chocolatey / Scoop / `gh` extension
  in addition to the static Go binary, so it slots into Linux, macOS,
  and Windows dev loops uniformly.

## Why it's interesting for AI / agent workflows

When an agent is editing a repo and CI keeps failing in some
combination of "lint job catches a nit, test matrix catches a flake,
release job catches a version bump", running the workflow under
`act` means the agent gets ground-truth pass/fail in the same
container the hosted runner uses, without burning a CI minute or
waiting for a remote queue. Combined with [`gh-dash`](../gh-dash/)
or [`gitu`](../gitu/) for PR-side feedback, `act` closes the loop:
edit → `act -j test` → fix → push only when green. Agents can also
use it to validate workflow YAML they author themselves before
opening a PR that touches `.github/`.

## When NOT to use it

Skip it for workflows that depend on hosted-only features — GitHub
App tokens minted via OIDC, the GitHub-hosted larger runners,
private network access from a self-hosted runner pool, or actions
that hard-code paths like `/home/runner/.cache/...`. Skip it when
the bottleneck is artifact upload / download to the GitHub Artifacts
service — `act`'s artifact server is a local stub. And skip it for
one-line workflows where the cost of pulling a 1+ GB runner image
outweighs the iteration speedup; just push and watch the hosted run.
