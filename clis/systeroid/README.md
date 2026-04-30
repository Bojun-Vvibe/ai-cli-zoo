# systeroid

- **Repo:** https://github.com/orhun/systeroid
- **Latest version:** v0.4.6 (2025-09-07)
- **HEAD on `main`:** `810348e`
- **License:** Apache-2.0 — [LICENSE-APACHE](https://github.com/orhun/systeroid/blob/main/LICENSE-APACHE)
  (dual-licensed MIT — [LICENSE-MIT](https://github.com/orhun/systeroid/blob/main/LICENSE-MIT))
- **Category:** sysadmin / kernel tuning

A **more powerful alternative to `sysctl(8)`** with embedded
documentation and an optional terminal UI (`systeroid-tui`). Lists,
greps, sets, and persists Linux kernel parameters with the same
ergonomics as `sysctl` but adds: built-in `--explain` that pulls the
kernel-tree documentation for any `kernel.*` / `vm.*` / `net.*` /
`fs.*` parameter, regex search across names *and* descriptions, a
TUI for browsing and editing, save / restore from JSON, and shell
completions out of the box. Written in Rust by `orhun`
(of `git-cliff`, `gpg-tui`, `kmon` fame), single static binary.

## Install

```bash
# Linux only (reads /proc/sys)

# Cargo
cargo install --locked systeroid systeroid-tui

# Arch (official repos)
pacman -S systeroid

# Nix
nix-env -iA nixpkgs.systeroid

# release tarball
curl -LO https://github.com/orhun/systeroid/releases/download/v0.4.6/systeroid-0.4.6-x86_64-unknown-linux-gnu.tar.gz
tar xf systeroid-0.4.6-x86_64-unknown-linux-gnu.tar.gz
sudo install systeroid systeroid-tui /usr/local/bin/

# verify
systeroid --version    # systeroid 0.4.6
```

## Sample usage

```bash
# list every parameter (drop-in for `sysctl -a`)
systeroid

# get one parameter (drop-in for `sysctl <key>`)
systeroid vm.swappiness

# set a parameter for the current boot
sudo systeroid -w vm.swappiness=10

# persist to /etc/sysctl.d/99-systeroid.conf
sudo systeroid -w vm.swappiness=10 --save

# the killer feature: explain a parameter using kernel docs
systeroid --explain vm.overcommit_memory
# prints the kernel-tree Documentation/admin-guide/sysctl/vm.rst entry
# (pre-fetch docs once with: systeroid --refresh-docs --kernel-docs <path>)

# regex search across names AND descriptions
systeroid --pattern 'tcp.*keepalive'

# launch the TUI for interactive browsing
systeroid-tui
# arrow keys to navigate, '/' to search, 'enter' to view docs, 'e' to edit

# dump current state for a host, restore on another
systeroid --output json > before.json
sudo systeroid --import before.json
```

## Niche it fills

The **"`sysctl` with docs and a TUI"** slot. `sysctl(8)` from
`procps-ng` is the POSIX answer and stays the right tool inside
shell scripts (it is everywhere, output is parseable, no surprises).
`systeroid` is what you reach for when you are *interactively*
tuning a kernel and you want the documentation for a parameter
without leaving the shell — `systeroid --explain
net.ipv4.tcp_tw_reuse` beats opening a browser tab to
kernel.org/doc, and the TUI beats `sysctl -a | grep` for discovery.

## Why it matters

- **`--explain` reads the in-tree kernel docs.** The same RST files
  that ship with the kernel source — no third-party blog post, no
  guesswork; you see the canonical description of what a parameter
  does and what units it takes.
- **Regex search over descriptions.** `systeroid --pattern
  'memory.*overcommit'` finds parameters by what they *do*, not just
  by name; `sysctl -a | grep` only matches the key.
- **TUI for the discovery loop.** When you are tuning a brand-new
  workload (database, kernel networking knobs, BPF JIT toggles), the
  edit-test-revert loop in `systeroid-tui` is far faster than
  remembering exact key names and re-typing `sysctl -w` each time.
- **JSON import / export for config-as-data.** Same shape on every
  host; pairs naturally with Ansible / Salt / NixOS for promoting a
  one-host experiment into fleet-wide config.

## Caveats

- **Linux only.** Reads `/proc/sys`; macOS / BSD `sysctl` semantics
  are different and out of scope.
- **`--explain` needs a one-time docs fetch.** Either point it at a
  local kernel source tree (`--kernel-docs /usr/src/linux`) or run
  `systeroid --refresh-docs` once with network access to populate
  the local cache; offline hosts need the cache pre-staged.
- **Persistence still writes to `/etc/sysctl.d/`.** `--save` is a
  convenience over `tee` — there is no rollback subsystem; back up
  the file before bulk edits.
- **Not a config-management tool.** For fleets, drive `systeroid
  --import` from your existing config-management layer; the tool
  itself does not know about hosts.
