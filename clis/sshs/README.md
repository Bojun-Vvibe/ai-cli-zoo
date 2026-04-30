# sshs

> **A terminal UI for SSH** — read your existing
> `~/.ssh/config`, present hosts in a fuzzy-searchable
> list, and connect with one keystroke. Pinned to
> **v4.7.2**
> ([LICENSE](https://github.com/quantumsheep/sshs/blob/main/LICENSE),
> MIT).

Source: <https://github.com/quantumsheep/sshs>

## TL;DR

`sshs` is the "I have 60 entries in my SSH config and
can't remember which is which" tool. Launch `sshs` and
it parses every `Host` block (including `Include`d
files), shows them in a Ratatui list with `Hostname`,
`User`, and any free-form comment, and lets you fuzzy
filter as you type. Hit Enter and it `exec`s `ssh
<host>` so you keep all of your existing
`ProxyJump` / `ForwardAgent` / `IdentityFile` /
`Match` semantics — `sshs` does not re-implement the
SSH client, it just drives it. Sort order, columns,
and search style are configurable via flags
(`--show-proxy-command`, `--sort-by-name`,
`--regex`), and selecting a host with Ctrl-X drops a
ready-to-paste `ssh ...` invocation back to your
shell instead of connecting, which is handy for
scripting or pasting into another pane.

## Install

```bash
# Homebrew
brew install sshs

# Cargo
cargo install sshs --locked

# prebuilt binaries
# https://github.com/quantumsheep/sshs/releases

# verify
sshs --version    # sshs 4.7.2
```

## Basic usage

```bash
# launch the picker (reads ~/.ssh/config + Includes)
sshs

# point at a different config file
sshs -c ./infra/ssh_config

# show ProxyCommand / ProxyJump in the list
sshs --show-proxy-command

# regex-filter hosts on launch
sshs --search 'prod-.*-eu'
```

## When to choose

- **You have a sprawling `~/.ssh/config`** —
  scrolling `cat ~/.ssh/config | grep Host` and
  retyping the alias gets old; sshs gives you a
  fuzzy picker without changing how SSH itself
  works.
- **You want to keep using OpenSSH semantics** —
  `ProxyJump`, agent forwarding, `Match`,
  `Include`, ssh-keysign all keep working because
  sshs shells out to the real `ssh`.
- **You teach others your jumphost setup** — the
  picker doubles as live documentation of which
  hosts exist and which user/proxy they need.

## When NOT to choose

- **You want a full session manager with saved
  panes / tmux integration** — use
  [`wishlist`](../wishlist/) or
  [`upterm`](../upterm/); sshs is a launcher, not
  a session daemon.
- **You don't use `~/.ssh/config`** — sshs
  derives its entire host list from that file (or
  one you point at). Without it there is nothing
  to pick.
- **You need to manage credentials / vault
  secrets** — use a secrets manager
  ([`bitwarden`](../bw/),
  [`pass`](../pass/), keyring); sshs only reads
  config metadata.

## Why it fits the zoo

Slots into the "lightweight terminal launcher over
an existing config file" cluster alongside
[`atuin`](../atuin/) (history),
[`zoxide`](../zoxide/) (cd), and
[`fzf`](../fzf/) (anything). sshs's specific gap is
**SSH host discovery from an unmodified
`~/.ssh/config`** — wishlist needs you to define
hosts in its own format; raw `fzf | xargs ssh` works
but loses the per-host metadata that sshs surfaces.

## Upstream pointers

- Repo: <https://github.com/quantumsheep/sshs>
- Release notes: <https://github.com/quantumsheep/sshs/releases>
- License: [MIT](https://github.com/quantumsheep/sshs/blob/main/LICENSE) (`LICENSE`)
- Maintainer: [@quantumsheep](https://github.com/quantumsheep)
