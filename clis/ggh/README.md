# ggh

> **Recall recent SSH sessions and reconnect with one
> keystroke** — a tiny Go CLI that reads `~/.ssh/config` plus
> the history of hosts you have actually `ssh`-ed into, lets
> you fuzzy-search both lists in a TUI picker, and re-launches
> the chosen connection (with the same user, port, and
> identity flags you used last time). Pinned to **v0.1.5**
> (commit `3797e998844d107c17590faff6412b8d5781b354`,
> [LICENSE](https://github.com/byawitz/ggh/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/byawitz/ggh>

## TL;DR

`ggh` ("git good… on hosts") is what you reach for when your
`~/.ssh/config` has fifty entries you barely remember, you
also keep `ssh ubuntu@10.0.4.117 -p 2222 -i ~/.ssh/id_homelab`
ad-hoc into IPs that never made it into the config, and the
recurring question is "wait, what was that box I sshed into
last Tuesday?". `ggh` keeps a local history file of every
session you have launched through it, and any time you want to
go back, you type `ggh` (no args), arrow-pick from the merged
config + history list, hit Enter, and you are reconnected with
the exact flags from last time.

## Install

```bash
# Homebrew (macOS / Linux)
brew install byawitz/tap/ggh

# Go (any OS with a Go toolchain)
go install github.com/byawitz/ggh@latest

# from a release binary (Linux x86_64 example)
curl -Lo ggh.tar.gz "https://github.com/byawitz/ggh/releases/download/v0.1.5/ggh_Linux_x86_64.tar.gz"
tar xf ggh.tar.gz && sudo install ggh /usr/local/bin/

# verify
ggh -v    # 0.1.5
```

`ggh` shells out to your system `ssh`, so `~/.ssh/config`,
agent forwarding, jump hosts, ProxyCommand, and ControlMaster
all keep working.

## License

Apache-2.0 — see [LICENSE](https://github.com/byawitz/ggh/blob/master/LICENSE).
Permissive, with explicit patent grant.

## One Concrete Example

```bash
# 1. ad-hoc connection (and recorded into history)
ggh ubuntu@10.0.4.117 -p 2222 -i ~/.ssh/id_homelab

# 2. reconnect to anything from history OR ~/.ssh/config (TUI picker)
ggh

# 3. fuzzy-filter the picker by substring
ggh homelab

# 4. list known hosts (config + history) without launching
ggh -l

# 5. show history only (chronological, with last-used timestamp)
ggh -h

# 6. show ~/.ssh/config entries only
ggh -c

# 7. wipe history (e.g. before screen-sharing)
ggh --clear
```

The history file lives at `~/.config/ggh/history.json` (one
JSON line per session: user, host, port, identity flags,
last-used timestamp).

## Niche It Fills

**SSH-history-aware reconnect picker.** The space has three
inhabitants: TUI pickers over `~/.ssh/config` only
([`sshs`](../sshs/), [`sshto`](https://github.com/vaniacer/sshto))
— no memory of ad-hoc connections you typed at the prompt;
shell-history grep (`Ctrl-R ssh`, [`atuin`](../atuin/),
[`mcfly`](../mcfly/), [`hstr`](../hstr/)) — works, but you must
remember the literal command you typed and you get the entire
shell history mixed in; and `ggh`, which is the only one that
both (a) records every `ssh`-as-launched-via-ggh into a
dedicated history file and (b) merges that history with
`~/.ssh/config` into a single picker. The killer property is
that ad-hoc IP-and-flags connections become first-class entries
you can reopen by name.

## Vs Already Cataloged

- **Vs [`sshs`](../sshs/):** sshs is a beautifully-rendered
  TUI picker over `~/.ssh/config` (and known_hosts). It does
  not remember ad-hoc `ssh user@1.2.3.4 -p 2222` connections
  unless you also added them to your config. `ggh` does. Use
  sshs when your config is the source of truth; use ggh when
  it isn't (and you can use both — they don't conflict).
- **Vs [`atuin`](../atuin/) / [`mcfly`](../mcfly/) /
  [`hstr`](../hstr/):** general shell-history search. They
  will find your `ssh ...` commands, but mixed with every
  `ls` and `git` you ever ran, and you must remember the
  literal command. `ggh` is purpose-built — its picker only
  shows ssh sessions, sorted and filterable by host.
- **Vs [`assh`](https://github.com/moul/assh) /
  [`storm`](https://github.com/emre/storm):** those are
  ssh-config *generators / managers* — they edit
  `~/.ssh/config` for you. `ggh` doesn't manage your config;
  it is the picker on top. Pair them: assh / storm to curate
  the config, ggh to remember everything you connected to
  outside it.
- **Vs [`tmuxinator`](../tmuxinator/) / [`sesh`](../sesh/):**
  those are session managers that, among other things, can
  open ssh windows. `ggh` is one keystroke narrower — pick
  one host, reconnect, done.

## Caveats

- **History is local-only.** No sync. If you wipe
  `~/.config/ggh/`, the history is gone. (Acceptable for the
  intended use; a dotfiles repo of the JSON file gives you
  trivial cross-machine sync if you want it.)
- **Only sessions launched *through* `ggh` are recorded.** A
  bare `ssh ...` typed at the prompt does not show up in
  `ggh`'s history (it does of course still work via your
  shell history). The mitigation most users adopt is
  `alias ssh=ggh` once they trust the workflow.
- **No agent / config-file editing.** `ggh` does not write
  to `~/.ssh/config`. If you want recurring hosts promoted
  from history to config, you do that yourself (or pair with
  `assh`/`storm`).
- **History file is plaintext JSON.** Hostnames, usernames,
  ports, and identity-file paths sit unencrypted in
  `~/.config/ggh/history.json`. Same threat model as your
  shell history, but worth knowing — the `--clear` flag
  exists for the screen-share moment.
- **Young project, narrow scope.** v0.1.x means the schema of
  `history.json` and the flag surface may still shift; the
  feature set is intentionally tiny (no multi-host fan-out,
  no session recording, no port-forward DSL). If you want a
  bigger workflow, [`sshs`](../sshs/) or
  [`sake`](../sake/) are different shapes.
