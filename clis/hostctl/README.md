# hostctl

> Snapshot date: 2026-05. Upstream: <https://github.com/guumaster/hostctl>

**Manage `/etc/hosts` like a CLI tool, not a `sudo $EDITOR` ritual.**
hostctl is a single Go binary that treats `/etc/hosts` as a
profile-grouped, version-controlled config file: you define named
profiles (`work`, `staging`, `client-acme`, `block-distractions`),
add hostnames to them, enable/disable a whole profile in one
command, and the underlying file stays human-readable with `# profile.<name>`
markers so non-hostctl edits and the tool's own writes coexist.

## Repo + version + license

- Repo: <https://github.com/guumaster/hostctl>
- Latest release: **`v1.1.4`** (2023-05-02)
- License: **MIT** —
  <https://github.com/guumaster/hostctl/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Go

## Install

```bash
# brew / scoop / docker / curl-bash all available
brew install guumaster/tap/hostctl

# Add three internal services under one toggleable profile
sudo hostctl add domains client-acme api.acme.local app.acme.local admin.acme.local --ip 127.0.0.1

# Disable the whole profile when you stop working on that client
sudo hostctl disable client-acme

# Block distractions during focus blocks (resolves to 0.0.0.0)
sudo hostctl add domains focus-mode news.ycombinator.com twitter.com reddit.com --ip 0.0.0.0
sudo hostctl enable focus-mode

# Sync a profile from a YAML/TOML/JSON file in your dotfiles repo
sudo hostctl replace --from ./hosts/staging.yaml staging
```

## Niche

The "**named profiles + enable/disable as the unit of work for
`/etc/hosts`**" slot. Where the default workflow is "open
`/etc/hosts` in `vim`, comment lines out, save", hostctl turns the
hosts file into a small database: list profiles, list domains in a
profile, enable / disable / replace whole profiles from CI or
dotfiles, and back up before each write. Useful for:

- **Multi-client local dev** — one profile per client, only one
  enabled at a time, no risk of `*.acme.local` resolving when you've
  switched to a different gig.
- **Distraction blocking** — a `focus-mode` profile that points
  social-media domains at `0.0.0.0`, toggled by a `pomo` /
  `timewarrior` / launchd hook.
- **Reproducible team setups** — a `staging.yaml` checked into the
  team dotfiles repo, applied with `hostctl replace --from
  staging.yaml staging` on every onboard.

## Why it matters

- **Profile-grouped writes** — every entry hostctl manages is
  bracketed by `# profile.begin <name>` / `# profile.end <name>`
  markers, so the file stays valid for any tool that reads it
  (DNS resolvers, container DNS shims, etc.) and your
  hand-maintained section at the top is left alone.
- **YAML / JSON / TOML / hosts-format import-export** —
  `hostctl replace --from file.yaml profile-name` makes the hosts
  file declarative; `hostctl backup` / `hostctl restore` give you
  point-in-time recovery.
- **Docker integration** — `hostctl add docker <container>`
  pulls IP + name from a running container so dev DNS for a Compose
  stack is one command, and it watches container restarts to keep
  the entry fresh.
- **Single Go binary** — no Python, no Node, no daemon; works the
  same on macOS / Linux / Windows / WSL, and the only privileged
  step is the actual write to `/etc/hosts` (which still needs
  `sudo`, by design — hostctl deliberately doesn't ship a setuid
  helper).
- **Project status caveat** — last release `v1.1.4` is 2023-05; the
  tool is feature-complete for its scope rather than abandoned, but
  there's no recent activity. The on-disk file format is stable, so
  switching away later is just `hostctl remove --all` + a manual
  cleanup.
