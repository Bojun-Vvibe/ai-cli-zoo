# nala

> **A friendlier front-end for `apt`** — same
> Debian / Ubuntu package operations as `apt`, but
> with a colourised columnar UI, parallel
> downloads from the fastest configured mirrors, a
> human-readable transaction history, and a real
> `nala history undo` that rolls a previous
> install / remove back. Pinned to **v0.16.1**
> ([LICENSE](https://gitlab.com/volian/nala/-/blob/main/LICENSE),
> GPL-3.0).

Source: <https://gitlab.com/volian/nala>

## TL;DR

`nala` is a Python wrapper around `python-apt`
(the same library `apt` itself uses) that
re-presents the package manager around three
ergonomics fixes: **(1) readable output** —
columns instead of a wall of `Inst foo (1.2.3
…)` lines, with grouped Install / Upgrade /
Remove / Downgrade / Reinstall sections and a
final summary; **(2) parallel mirror fetches** —
`nala fetch` benchmarks the official mirror list
and writes the top N into
`/etc/apt/sources.list.d/nala-sources.list`,
then `nala update` / `nala install` pulls from
all of them in parallel via `aria2c`-style
range requests; **(3) a real transaction log** —
every operation lands in `/var/lib/nala/history.json`
with a numeric ID, and `nala history undo <id>` /
`nala history redo <id>` re-applies the inverse
transaction the same way `dnf history undo` does
on Fedora. The CLI mirrors `apt` verb-for-verb
(`install` / `remove` / `purge` / `update` /
`upgrade` / `search` / `show` / `list` / `clean`
/ `autoremove`), so muscle-memory transfers
1:1 — `alias apt=nala` is the documented adoption
path.

## Install

```bash
# Debian 12+ / Ubuntu 23.04+ (in main repos)
sudo apt install nala

# Older releases — official Volian repo
echo "deb https://deb.volian.org/volian/ scar main" \
  | sudo tee /etc/apt/sources.list.d/volian-archive-scar-unstable.list
wget -qO- https://deb.volian.org/volian/scar.key \
  | sudo tee /etc/apt/keyrings/volian-archive-scar-unstable.gpg > /dev/null
sudo apt update && sudo apt install nala

# verify
nala --version    # nala 0.16.1
```

## Basic usage

```bash
# pick the fastest mirrors and write a sources file
sudo nala fetch --auto -y

# refresh + upgrade with the readable UI
sudo nala update
sudo nala upgrade

# install / remove / purge
sudo nala install ripgrep fd-find
sudo nala remove ripgrep
sudo nala purge fd-find

# search and show
nala search '^rust-'
nala show ripgrep

# transaction history + undo
nala history
sudo nala history undo 42
sudo nala history redo 42
sudo nala history clear --all
```

## When to choose

- **You live on Debian / Ubuntu and stare at `apt`
  output every day** — nala is a strict superset
  of `apt`'s read surface, so the upgrade is
  zero-cost: same package set, same lock file,
  same `dpkg` underneath, but readable.
- **Your fleet pulls from a slow default mirror**
  — `nala fetch` is the fastest path to "use the
  three closest mirrors in parallel" without
  hand-editing `sources.list`.
- **You want `dnf history undo` on Debian** —
  this is the only mainstream apt front-end that
  records inverse transactions and replays them.

## When NOT to choose

- **You're on Fedora / Arch / Alpine** — wrong
  substrate; this only wraps `apt`.
- **You're scripting unattended upgrades in CI**
  — stick with `apt-get` / `unattended-upgrades`;
  nala's pretty UI is wasted on `set -e` runners
  and adds a Python dependency.
- **You need reproducible declarative package
  state** — use `nix`, `apt-file` plus a manifest,
  or a config-management tool; nala is still
  imperative `apt`.

## Why it fits the zoo

Sits next to other "modern re-skins of a
universal Unix verb" entries —
[`bat`](../bat/) for `cat`, [`eza`](../eza/) for
`ls`, [`dust`](../dust/) for `du`, [`procs`](../procs/)
for `ps`, [`duf`](../duf/) for `df`. nala is the
package-manager-shaped member of that family:
a drop-in that respects existing muscle memory
while fixing the readability and history gaps
the original verb never closed.

## Upstream pointers

- Repo: <https://gitlab.com/volian/nala>
- Release notes: <https://gitlab.com/volian/nala/-/releases>
- License: [GPL-3.0](https://gitlab.com/volian/nala/-/blob/main/LICENSE)
- Docs: <https://gitlab.com/volian/nala/-/wikis/home>
- Maintainer: [Volian](https://gitlab.com/volian)
