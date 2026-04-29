# wishlist

> **The SSH directory** — a single Go binary from
> Charmbracelet that reads a YAML / `~/.ssh/config`
> list of hosts and presents them as a Bubble Tea
> TUI you connect to with one keystroke, optionally
> exposing the same directory over SSH itself so a
> shared bastion lets the team `ssh wishlist.local`
> and pick from a curated menu. Pinned to
> **v0.15.2**
> ([LICENSE](https://github.com/charmbracelet/wishlist/blob/main/LICENSE),
> MIT, © Charmbracelet, Inc.).

Source: <https://github.com/charmbracelet/wishlist>

## TL;DR

`wishlist` is the smallest reasonable answer to
"my team has 40 hosts and `~/.ssh/config` is
unreadable on a phone in a hotel room". It reads
hosts from one of three sources — a `wishlist.yaml`
file, a Wishlist-flavoured `~/.ssh/config` (it
honours `Host`, `HostName`, `User`, `Port`,
`IdentityFile`, `IdentitiesOnly`,
`PreferredAuthentications`, `ForwardAgent`,
`RemoteCommand`, `SendEnv`, `SetEnv`, plus its
own `Link` / `ConnectTimeout` extensions), or an
HTTP endpoint returning YAML — and renders them in
a Bubble Tea TUI: arrow keys to pick, `enter` to
connect, `esc` to come back to the list when the
session ends. Two run modes: **local** (`wishlist`
on your laptop, just a nicer host picker) and
**server** (`wishlist serve` on a bastion or jump
host, which exposes the same directory as an
SSH-listening service so anybody can
`ssh -p 2222 bastion.example.com` and land in the
TUI without needing a local install). The server
mode terminates incoming SSH with a host key under
`.wishlist/`, authenticates either against
`authorized_keys` files referenced by each entry
or against an upstream OIDC / GitHub / GitLab
team, and proxies the chosen connection through
itself — agent forwarding included if the entry
asks for it. There is also a `factory`
extensibility hook (you write a tiny Go program
that imports `wishlist` as a library and yields
an `[]*wishlist.Endpoint`) for the case where the
list of hosts is dynamic — pulled from cloud
inventory, a CMDB, or a service registry — and
you would rather generate it than hand-edit YAML.

## Install

```bash
# Homebrew (macOS / Linux)
brew install charmbracelet/tap/wishlist

# Go (any platform with a Go toolchain)
go install github.com/charmbracelet/wishlist/cmd/wishlist@latest

# Nix
nix-env -iA nixpkgs.wishlist

# Arch Linux (AUR)
yay -S wishlist

# Debian / Ubuntu via Charm apt repo
echo 'deb [trusted=yes] https://repo.charm.sh/apt/ /' | \
  sudo tee /etc/apt/sources.list.d/charm.list
sudo apt update && sudo apt install wishlist

# Fedora / RHEL via Charm yum repo
echo '[charm]
name=Charm
baseurl=https://repo.charm.sh/yum/
enabled=1
gpgcheck=0' | sudo tee /etc/yum.repos.d/charm.repo
sudo yum install wishlist

# Docker
docker run --rm -it -v $PWD:/wishlist \
  charmcli/wishlist:v0.15.2

# Pre-built tarballs / .deb / .rpm / Windows zip
# from https://github.com/charmbracelet/wishlist/releases

# verify
wishlist --version    # wishlist version 0.15.2
```

## Basic usage

```bash
# 1. Local mode: read ~/.ssh/config (or
#    .wishlist/config.yaml in $PWD) and pick a host.
wishlist

# 2. Point at an explicit YAML
cat > hosts.yaml <<'YAML'
endpoints:
  - name: prod-app-1
    address: 10.0.1.10:22
    user: deploy
    forward_agent: true
  - name: prod-db
    address: db.internal:22
    user: dba
    identity_files:
      - ~/.ssh/id_ed25519_dba
  - name: dev-shell
    address: dev.example.com
    user: hacker
    remote_command: tmux new -A -s dev
YAML
wishlist --config hosts.yaml

# 3. Use a Wishlist-flavoured ssh_config
wishlist --config ~/.ssh/config

# 4. Server mode: expose the directory over SSH.
#    Anyone who can reach :2222 lands in the TUI;
#    `auth` controls who is let in.
wishlist serve --port 2222 --config hosts.yaml \
  --host-key .wishlist/server_ed25519

# 5. Server mode with GitHub team auth — only members
#    of the listed team see the menu.
wishlist serve --port 2222 \
  --config hosts.yaml \
  --auth github://my-org/sre-team

# 6. From a client, connect to the bastion and get
#    the picker — no local wishlist install needed.
ssh -p 2222 bastion.example.com

# 7. Programmatic (factory) hosts list — generate
#    endpoints from cloud inventory in Go and embed
#    wishlist as a library; see ./_examples/factory
#    in the upstream repo for a complete sample.
```

## When to choose

- **Your team's `~/.ssh/config` has crossed the
  "I cannot remember which one is staging" line** —
  `wishlist` turns the file you already have into a
  picker, with no migration. `Host` / `HostName` /
  `User` / `Port` / `IdentityFile` blocks render
  straight through.
- **You run a bastion / jump host and want a menu
  on it** — `wishlist serve` is the missing UX:
  one SSH endpoint, a TUI, and per-entry
  `authorized_keys` / OIDC / GitHub-team gating,
  without writing a wrapper script around `ssh -t
  user@bastion …`.
- **Your host list is generated, not hand-written**
  — the `factory` extension point lets you import
  `wishlist` as a Go package and feed it a
  programmatically-built `[]*Endpoint`, so the
  picker stays in sync with your cloud inventory
  without templating YAML.
- **You want session affordances inline** — entries
  can specify `remote_command` (drop straight into
  `tmux new -A -s …`), `forward_agent`,
  `send_env` / `set_env`, and `local_forward` /
  `remote_forward` per host, so the picker can
  also encode the conventions a team has built up.
- **You like the rest of the Charm stack** —
  visually consistent with [`gum`](../gum/),
  [`glow`](../glow/), [`vhs`](../vhs/),
  [`soft-serve`](../soft-serve/),
  [`mods`](../mods/), built on the same
  Bubble Tea / Lip Gloss runtime, so terminal
  rendering, mouse support, and keybinding
  conventions match your existing Charm tools.

## When NOT to choose

- **You are a single user with three machines** —
  plain `ssh prod` / `ssh dev` is fine; a TUI on
  top of three entries is overhead, not ergonomics.
- **You need a full SSH client replacement** —
  `wishlist` is a *picker* in front of OpenSSH (or
  its own embedded `golang.org/x/crypto/ssh`
  client in server mode); features like `ProxyJump`
  chains beyond what the YAML / ssh_config grammar
  expresses, dynamic SOCKS forwarding, control
  sockets, or scriptable interactive sessions
  belong with `ssh` proper, [`mosh`](https://mosh.org),
  or [`autossh`](https://www.harding.motd.ca/autossh/).
- **Your access model is "everybody is root via
  password"** — `wishlist serve` strongly assumes
  public-key auth (or upstream OIDC / GitHub /
  GitLab team checks). Password-only flows are not
  the design centre.
- **You need a graphical SSH client UX** with
  saved-passwords-per-tab, file-browser side
  panel, and SFTP visualisation — that is
  Termius / SecureCRT / Royal TSX territory.
  `wishlist` is text-mode and proud of it.
- **You want centralised audit logging of every
  command typed through the bastion** — `wishlist
  serve` proxies sessions but is not a session
  recorder. Pair it with [Teleport](https://goteleport.com/)
  or [`sshrec`](https://github.com/dnaeon/sshrec)
  if compliance requires keystroke-level capture.

## Why it fits the zoo

The zoo's terminal-application cluster
([`gum`](../gum/),
[`glow`](../glow/),
[`vhs`](../vhs/),
[`mods`](../mods/),
[`soft-serve`](../soft-serve/),
[`gitui`](../gitui/),
[`lazygit`](../lazygit/),
[`k9s`](../k9s/),
[`atuin`](../atuin/)) collects the
single-binary, opinionated TUIs that have replaced
hand-rolled shell scripts for everyday operator
tasks. `wishlist` is the SSH-directory entry in
that family, sister to
[`soft-serve`](../soft-serve/) (Charm's
SSH-served self-hostable Git server): both turn
SSH itself into the application transport, both
ship as one Go binary, both speak the same Charm
TUI vocabulary, and both are practical to host on
a small VPS as part of a "everything-over-SSH"
team setup.

## Upstream pointers

- Repo: <https://github.com/charmbracelet/wishlist>
- Release notes: <https://github.com/charmbracelet/wishlist/releases>
- License:
  [MIT](https://github.com/charmbracelet/wishlist/blob/main/LICENSE)
  (© Charmbracelet, Inc.)
- Examples directory:
  <https://github.com/charmbracelet/wishlist/tree/main/_examples>
  — covers `factory`, multiple-config, custom-auth,
  and Docker-based deployments.
- Sister projects from the same vendor that pair
  with `wishlist`: [`soft-serve`](../soft-serve/)
  (SSH-served Git host), [`gum`](../gum/)
  (shell-script-grade TUI primitives),
  [`glow`](../glow/) (Markdown viewer),
  [`charm`](https://github.com/charmbracelet/charm)
  (encrypted cloud KV / fs / identity stack).
- Maintainers:
  [@charmbracelet](https://github.com/charmbracelet)
  team (Toby Padilla, Christian Rocha, et al.).
