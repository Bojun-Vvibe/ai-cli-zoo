# hut

> **Single-binary CLI for the sourcehut suite** — talks
> GraphQL to every sourcehut service (`git.sr.ht`,
> `hg.sr.ht`, `builds.sr.ht`, `lists.sr.ht`,
> `todo.sr.ht`, `paste.sr.ht`, `pages.sr.ht`,
> `meta.sr.ht`, `man.sr.ht`) from one consistent verb
> surface, with offline OAuth2-token auth, JSON output for
> shell pipelines, and a `hut builds submit` that turns a
> local manifest plus the current `git` HEAD into a CI run
> in one shell line. Pinned to **v0.8.0**
> ([LICENSE](https://git.sr.ht/~xenrox/hut/tree/master/item/LICENSE),
> AGPL-3.0-only).

Source: <https://git.sr.ht/~xenrox/hut>

## TL;DR

[sourcehut](https://sourcehut.org/) is a federation of
small focused services rather than one monolithic forge —
each surface (git hosting, mailing lists, build jobs, bug
tracker, paste bin, static pages) is its own subdomain
with its own GraphQL API. `hut` is the **one CLI that
speaks to all of them** with the same `<service>
<noun> <verb>` shape: `hut git list`, `hut git
clone <user>/<repo>`, `hut builds submit ./.build.yml`,
`hut builds list`, `hut builds log <id>`, `hut paste
create < file`, `hut paste list`, `hut todo ticket
create <tracker>`, `hut lists subscribe <list>`, `hut
meta show`, `hut export` (a full account dump for
backup or migration), `hut import` (replay that dump
elsewhere). One `hut init` walks you through generating a
personal-access OAuth2 token in a browser and stores it
under `$XDG_CONFIG_HOME/hut/config`; subsequent commands
read the token from that file with no further interactive
prompt — works the same in CI, in `cron`, and over SSH
on a headless server. `--format json` is implemented for
every list-shaped command, so the output composes with
[`jq`](../jq/) the same way `gh` output does (e.g.
`hut builds list -j | jq -r '.results[] | select(.status
=="FAILED") | .id'`). For build jobs in particular, `hut
builds submit` is the everyday verb: a single
`.build.yml` manifest in the repo (`image`, `packages`,
`sources`, `tasks`) becomes a queued CI job against the
current branch with one shell line, log streaming
included, exit-status mirrored locally, no web-UI round
trip needed. The official CLI of the project, written and
maintained by the sourcehut team alongside the services
themselves, so the API surface stays in lock-step
with what `hut` exposes.

## Install

```bash
# Arch Linux (AUR)
yay -S hut

# Alpine (community)
apk add hut

# Pre-built tarballs from the project's release page
curl -LO https://git.sr.ht/~xenrox/hut/refs/v0.8.0/hut-v0.8.0.tar.gz
tar xzf hut-v0.8.0.tar.gz
cd hut-v0.8.0 && make && sudo make install

# Build from source (Go 1.20+; scdoc for man pages)
git clone --branch v0.8.0 https://git.sr.ht/~xenrox/hut
cd hut && make && sudo make install
```

## Usage

```bash
# One-time auth — opens a browser to mint a personal
# OAuth2 token, stores it in ~/.config/hut/config
hut init

# Git
hut git list                              # your repos
hut git clone ~user/repo                  # ssh clone shorthand
hut git create new-project --visibility unlisted

# Build jobs (the everyday workhorse)
cat > .build.yml <<'YAML'
image: alpine/edge
packages: [go, make]
sources:
  - https://git.sr.ht/~me/myproject
tasks:
  - test: |
      cd myproject
      go test ./...
YAML
hut builds submit .build.yml              # streams logs to your TTY
hut builds list                           # recent runs
hut builds log <id>                       # full log of a past run
hut builds resubmit <id>                  # rerun

# Pastes
echo "throwaway" | hut paste create --visibility unlisted
hut paste list

# Bug tracker
hut todo ticket create ~me/myproject -t "auth fails on iOS"
hut todo ticket list   ~me/myproject

# Mailing lists
hut lists subscribe ~sircmpwn/sr.ht-discuss
hut lists list

# Account-wide backup / migration
hut export ./sourcehut-backup/
hut import ./sourcehut-backup/            # replays into a new account
```

## Why it's interesting

The "single CLI for a hosted forge" slot has two
incumbents — [`gh`](../gh/) for GitHub and
[`glab`](../glab/) for GitLab — and one obvious gap:
sourcehut, which is structurally different from both
because it is a federation of small services rather than a
monolith. `hut` is the official answer to that gap and the
only client that talks to every sourcehut service from one
binary; the *unofficial* alternatives (`hubsrht`,
`srht-cli` forks, hand-rolled `curl` against the GraphQL
endpoints) cover one or two services at most and lag the
API. Pick `hut` when (a) your project actually lives on
sourcehut and you want the equivalent of `gh pr view` /
`gh run watch` for `git.sr.ht` + `builds.sr.ht`, (b) you
need scriptable CI submission from a laptop without
pushing a branch first — `hut builds submit` queues a job
against your local manifest and current HEAD, useful for
testing changes that you don't want in the repo's branch
history, or (c) you need a clean `export`/`import` round
trip across sourcehut accounts (moving from the hosted
`sr.ht` instance to a self-hosted one is a real
migration path that `hut export ./bak/ && hut --instance
mysrht.example.org import ./bak/` makes one-shot). Not
the right pick if you don't use sourcehut — there is no
GitHub/GitLab story here, by design, and the AGPL-3.0
license means embedding `hut` into a hosted product
carries source-disclosure obligations on the hosting side.
Maintained by the sourcehut team, on a steady release
cadence since 2021, with v0.8.0 (the current tag) adding
finer-grained `builds` querying, expanded `meta` (PGP
keys, SSH keys, account profile) coverage, and a stable
JSON output schema across every list verb.
