# gickup

> **Declarative cross-forge git repository backup tool** — one
> static Go binary that reads a YAML config listing source forges
> (GitHub, GitLab, Gitea, Bitbucket, Forgejo, Sourcehut, OneDev,
> raw `git` URLs) and destination forges (any of the above, plus
> local directory, S3, or a chained mirror), discovers every
> repo the configured tokens can see (user, org, starred, watched,
> arbitrary include/exclude regex), and `git clone --mirror`s each
> one to every destination on a schedule — covering branches,
> tags, LFS, wikis, issues, releases, and pull/merge requests
> where the destination supports them. Pinned to **v0.10.42**
> (released 2026-05-04,
> [`gh api repos/cooperspencer/gickup/releases/latest`](https://github.com/cooperspencer/gickup/releases/latest),
> [LICENSE](https://github.com/cooperspencer/gickup/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/cooperspencer/gickup>

## TL;DR

"Back up every repo I own across every forge to somewhere I
control" is a problem with three bad historical answers: a
hand-rolled `for repo in $(gh repo list ...); do git clone
--mirror; done` script (works once, rots immediately when a
token expires or a new repo is added), per-forge official
exporters (GitHub Enterprise's `ghe-migrator`, GitLab's
project-export endpoint, Gitea's dump command — each speaks one
forge and produces a different artefact shape), or a heavy
GitOps platform (Bacula / Borg over a clone tree — captures
*files* not *forge state*). `gickup` is the focused middle: a
single declarative `conf.yml` lists sources and destinations,
`gickup conf.yml` runs the cycle once, and the same binary
under cron / systemd-timer / `--cron` mode runs it on a
schedule. Sources discover repos through the forge's own API
(token-scoped: a GitHub PAT sees personal + org repos it has
access to; include/exclude regex narrow the set), destinations
push a `git push --mirror` so refs match exactly (no orphaned
branches, no missing tags). The killer move is **N×M decoupling**:
add a new source forge by adding one block, add a new destination
mirror by adding one block, and every existing source-destination
pair light up automatically — so a "GitHub + GitLab → local NAS
+ self-hosted Gitea + S3 cold storage" setup is one config file,
not five custom scripts.

## Install

```bash
# Pre-built binary from a release (Linux / macOS / Windows)
curl -L \
  https://github.com/cooperspencer/gickup/releases/download/v0.10.42/gickup_linux_amd64.tar.gz \
  | tar xz && sudo mv gickup /usr/local/bin/

# Docker (recommended for cron-style scheduled runs)
docker run --rm -v "$PWD/conf.yml:/conf.yml:ro" -v /backup:/backup \
  cooperspencer/gickup /conf.yml

# Go (any platform with a Go toolchain)
go install github.com/cooperspencer/gickup@latest

# verify
gickup --version
```

## Representative examples

```yaml
# conf.yml — back up GitHub user + org to a local mirror tree
# and push to a self-hosted Gitea instance
source:
  github:
    - user: alice
      token: $GITHUB_TOKEN
      starred: true
      include: ["^infra-.*", "^docs$"]
      exclude: ["^archive-.*"]
destination:
  local:
    - path: /backup/git-mirror
      structured: true        # owner/repo.git layout
      lfs: true
      bare: true
  gitea:
    - url: https://gitea.internal
      token: $GITEA_TOKEN
      user: backup-bot
      mirror: true
      mirrorinterval: 8h0m0s
```

```bash
# 1. One-shot run
gickup conf.yml

# 2. Run on a schedule (built-in cron expression)
gickup --cron "0 */6 * * *" conf.yml

# 3. Dry-run: list what would be cloned without touching disk
gickup --dry conf.yml

# 4. Quiet mode for cron — only log errors
gickup --quiet conf.yml

# 5. Healthcheck ping after a successful cycle
gickup --healthchecks https://hc-ping.com/<uuid> conf.yml
```

## When to use vs. alternatives

- Pick **gickup** when the requirement is "every repo across
  multiple forges, mirrored to multiple destinations, on a
  schedule, declared in one YAML" and the destinations include
  at least one *git forge* (not just a tarball dump) so the
  mirror is itself browsable / cloneable.
- Pick `git clone --mirror` in a hand-rolled script when there
  is exactly one source and one destination and the repo list
  is static — gickup's value is the N×M discovery, not the
  underlying git plumbing.
- Pick **`gh repo clone`** / **`glab repo clone`** for ad-hoc
  one-time bulk pulls — those don't track new repos and don't
  push to a mirror.
- Pick a forge-native exporter (GitLab project export,
  `gitea dump`, GitHub Enterprise `ghe-migrator`) when the goal
  is *migrate* (one-way, with issues/PRs/users/permissions)
  rather than *mirror* (continuous, refs-only by default) —
  gickup can mirror issues/PRs where destinations support it,
  but a true migration tool wins on identity preservation.
- Pick [`restic`](../restic/) / [`borgbackup`](../borgbackup/) /
  [`kopia`](../kopia/) for the orthogonal layer: filesystem
  snapshot + dedup + retention of the entire `/backup/git-mirror`
  tree gickup produces, so a corrupted mirror push can be rolled
  back to yesterday's state. Compose: gickup populates the tree,
  restic snapshots it.
- Caveats: pre-1.0 schema (occasionally adds non-trivial config
  keys between minors — pin the binary version in CI), token
  scopes are the gickup user's responsibility (a token without
  `repo` scope silently produces an empty mirror — `--dry`
  exposes this early), and mirror destinations that disable
  force-push will refuse rewritten history (which is the safe
  default — opt in per-destination only when you mean it).
