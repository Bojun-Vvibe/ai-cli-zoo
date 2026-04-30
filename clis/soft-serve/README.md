# soft-serve

> **A self-hostable Git server for the command line.** A single
> Go binary from Charm that runs an SSH-fronted Git server with
> a built-in TUI you reach by `ssh git.example.com` — push,
> pull, browse repos, read READMEs (rendered with the same
> Charm markdown stack as [glow](../glow/)), set per-repo
> access, all without a web frontend or a database. Pinned to
> **v0.11.6**
> ([LICENSE](https://github.com/charmbracelet/soft-serve/blob/main/LICENSE),
> MIT).

Source: <https://github.com/charmbracelet/soft-serve>

## TL;DR

`soft-serve` is the right answer when you want a private Git
remote on a small VM (or a NAS, or a homelab box) without
running Gitea / Forgejo / GitLab and without exposing HTTP. It
listens on SSH (port 23231 by default), serves Git over SSH the
normal way (`git clone ssh://user@host:23231/repo`), and
*also* renders an interactive TUI to that same SSH endpoint so
`ssh host` (no command) drops you into a repo browser with
file tree, commit log, and rendered README. Auth is SSH public
keys; admin is one user with the `admin: true` flag in the
config; everything is files on disk under `~/.soft-serve/` so
backups are `tar`. For an LLM-CLI workflow, it is the smallest
possible "real Git remote" you can stand up to host the
prompt / agent-config / eval-dataset repos that you do not
want on a public forge.

## Install

```bash
# Homebrew
brew install charmbracelet/tap/soft-serve

# Go
go install github.com/charmbracelet/soft-serve/cmd/soft@latest

# Linux package managers (Charm apt/yum repos)
# Debian/Ubuntu:
#   sudo mkdir -p /etc/apt/keyrings
#   curl -fsSL https://repo.charm.sh/apt/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/charm.gpg
#   echo "deb [signed-by=/etc/apt/keyrings/charm.gpg] https://repo.charm.sh/apt/ * *" | sudo tee /etc/apt/sources.list.d/charm.list
#   sudo apt update && sudo apt install soft-serve
# Arch: pacman -S soft-serve
# Nix:  nix-env -iA nixpkgs.soft-serve

# Docker
docker run -d --name soft-serve \
  -p 23231:23231 -p 23232:23232 -p 9418:9418 -p 9419:9419 \
  -v $HOME/soft-serve:/soft-serve \
  charmcli/soft-serve:latest

# from a release tarball
curl -Lo soft-serve.tar.gz "https://github.com/charmbracelet/soft-serve/releases/download/v0.11.6/soft-serve_0.11.6_Darwin_arm64.tar.gz"
tar xf soft-serve.tar.gz
sudo install soft-serve_0.11.6_Darwin_arm64/soft /usr/local/bin/

# verify + first run
soft --version           # soft version v0.11.6
SOFT_SERVE_INITIAL_ADMIN_KEYS="$(cat ~/.ssh/id_ed25519.pub)" soft serve
```

First run materialises `~/.soft-serve/` with a default
`config.yaml`, an SSH host key, and an empty repo store. The
key in `SOFT_SERVE_INITIAL_ADMIN_KEYS` becomes the bootstrap
admin; everything else is configured by `ssh -p 23231 host
settings ...`.

## License

MIT — see
[LICENSE](https://github.com/charmbracelet/soft-serve/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. start the server (foreground; pair with systemd / launchd in prod)
SOFT_SERVE_INITIAL_ADMIN_KEYS="$(cat ~/.ssh/id_ed25519.pub)" \
  soft serve

# 2. from your laptop, create a repo and push a prompt library
ssh -p 23231 admin@git.example.com repo create prompts --description "house prompts"
git remote add house ssh://admin@git.example.com:23231/prompts
git push house main

# 3. browse via TUI (no command after the host)
ssh -p 23231 admin@git.example.com
#   → opens repo list, arrow keys navigate, `?` for help,
#     enter on a repo opens commit log + file tree + README

# 4. give a teammate read-only access to one repo
ssh -p 23231 admin@git.example.com user create alice --key "$(cat alice.pub)"
ssh -p 23231 admin@git.example.com repo collab add prompts alice read-only

# 5. mirror a public repo so the agent CLI clones from inside the VPC
ssh -p 23231 admin@git.example.com repo import \
  https://github.com/some/public-evals.git evals --mirror

# 6. expose Git over the unauthenticated git:// protocol for read-only mirrors
#    (port 9418, set anon-access: read in config.yaml)
git clone git://git.example.com:9418/evals

# 7. webhook on every push (for CI triggers / agent re-indexing)
ssh -p 23231 admin@git.example.com repo webhook create prompts \
  https://ci.example.com/hooks/git-push --content-type json
```

## Niche It Fills

**Smallest viable private Git remote that still feels like a
forge.** Bare `git init --bare` over SSH works but gives you
no UI, no per-repo permissions, no webhooks, no README
rendering. Gitea / Forgejo / GitLab give you all of that plus
a database, a web server, an admin UI, and a 200MB+ memory
floor. `soft-serve` is the middle: one Go binary, one
directory, one open port, and you still get a usable
repo-browser TUI, SSH-key auth, per-repo collaborators, and
webhooks. Right-sized for a homelab, an air-gapped lab box, or
the "private agent-config repo" that should not live on a
public forge.

## Why use it

Three things `soft-serve` does that bare Git over SSH does
not, that pay back the switching cost:

1. **TUI front-end at the same SSH endpoint.** `ssh
   git.example.com` (no command) drops into the repo browser;
   commit log, file tree, README rendered with Charm's
   markdown stack, all over the same port that serves Git.
   No web server to operate, no extra port to firewall.
2. **Real per-repo access control with SSH keys.** Users are
   created by uploading their public key
   (`ssh ... user create alice --key ...`); per-repo
   collaborators get `read-only` / `read-write` / `admin`
   roles. Bare Git over SSH gives you "everyone with a shell
   account on the box can push everywhere".
3. **Webhooks + LFS + mirroring out of the box.** `ssh ...
   repo webhook create` fires on push events the way a forge
   does, so an agent CLI's "re-index prompts repo on every
   push" loop is a one-liner. `repo import --mirror` keeps a
   local read-only mirror of an upstream public repo, useful
   for agent CLIs that want to clone evals / datasets from
   inside a VPC.

For an LLM-CLI workflow, `soft-serve` is the answer to "where
do the prompt libraries, agent configs, eval datasets, and
shared `~/.config/<agent>/` snippets live when GitHub /
GitLab is not allowed?" — one VM, one binary, real Git, real
ACLs, no database.

## Vs Already Cataloged

- **Vs [`gitui`](../gitui/) / [`lazygit`](../lazygit/):**
  orthogonal. Those are *client-side* TUIs that drive a local
  Git working copy. `soft-serve` is the *server*. A typical
  setup uses `lazygit` on the laptop and pushes to a
  `soft-serve` remote.
- **Vs Gitea / Forgejo / GitLab (not cataloged):** the heavy
  forges win on web UI, issue tracker, CI integration, and
  pull requests. `soft-serve` deliberately has none of those —
  no issues, no PRs, no web UI. Pick a forge when humans need
  to file bugs and review code in a browser; pick `soft-serve`
  when the only consumers are CLIs (developers + agents) and
  you want one binary instead of a Postgres + web stack.
- **Vs bare `git init --bare` over SSH (not cataloged):**
  baseline. Zero ops cost, zero features. `soft-serve` adds
  the TUI browser, per-repo ACLs, webhooks, mirroring, and
  config-as-code without forcing a forge on you.
- **Vs [`wishlist`](../wishlist/) (Charm sibling):** different
  tool. `wishlist` is an SSH directory ("which host am I
  jumping to today?"); `soft-serve` is an SSH-served Git host.
  They compose: a `wishlist` entry can point at a `soft-serve`
  endpoint.

## Caveats

- **No issues, no PRs, no web UI.** Deliberate. If your team's
  workflow needs in-browser code review, `soft-serve` is the
  wrong tool — reach for Gitea / Forgejo / a hosted forge.
- **Single-node only.** No HA story, no clustering. State
  lives in `~/.soft-serve/`; HA is "back up the directory and
  restore it on the failover node".
- **Config split between `config.yaml` and live SSH commands.**
  Bootstrap config is a YAML file; runtime mutations (create
  repo, add user, set webhook) are SSH subcommands that mutate
  the same on-disk store. Reading the current state means
  `ssh host repo list` etc., not "cat one config file"; manage
  expectations.
- **Default ports are non-standard.** SSH on 23231 (not 22) so
  it can coexist with the host's real `sshd`; Git protocol on
  9418; HTTP / stats on 23232 / 9419. Firewall rules and
  client `~/.ssh/config` entries need explicit `Port`.
- **Markdown rendering is the Charm stack.** Same renderer as
  `glow` / `gum format`; great for typical READMEs, occasional
  drift on exotic markdown extensions (Mermaid blocks render
  as code, not diagrams). Match expectations to a terminal
  reader, not a browser.
