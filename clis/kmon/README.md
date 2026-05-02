# kmon

> **A Linux kernel manager and activity monitor in a single TUI** —
> browse loaded modules, load/unload/blacklist them, watch real-time
> kernel ring-buffer (`dmesg`) output, and inspect module dependencies
> and used-by trees, all without leaving the terminal.
> Pinned to **v1.7.1**
> ([LICENSE](https://github.com/orhun/kmon/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/orhun/kmon>

## TL;DR

`kmon` is a Rust+`tui-rs` TUI that wraps the standard kernel module
toolchain (`lsmod`, `modinfo`, `modprobe`, `insmod`, `rmmod`,
`depmod`, `dmesg`) into one keyboard-driven dashboard. The left
pane is the live module list (filterable, sortable by name / size
/ used-by count). The right pane shows `modinfo` for the selected
module — author, license, parameters, dependencies, signed-or-not.
The bottom pane streams the kernel log so you immediately see what
a `load` or `unload` action printed. Common actions are one
keystroke: `l` to load, `u` to unload, `b` to blacklist, `r` for
the dependency tree, `/` to filter, `e` to edit module parameters.

## Install

```bash
# cargo (any platform with Rust)
cargo install kmon

# Homebrew
brew install kmon

# Linux package managers
# Arch:           pacman -S kmon
# Alpine:         apk add kmon
# Nix:            nix-env -iA nixpkgs.kmon
# Void:           xbps-install kmon
# Debian/Ubuntu:  download .deb from the GitHub release page

# single-file binary release
curl -LO https://github.com/orhun/kmon/releases/download/v1.7.1/kmon-1.7.1-x86_64-unknown-linux-gnu.tar.gz
tar xf kmon-1.7.1-x86_64-unknown-linux-gnu.tar.gz
sudo install -m755 kmon /usr/local/bin/

# verify
kmon --version    # kmon 1.7.1

# run (root needed for load/unload; read-only mode works as user)
sudo kmon
```

`kmon` is **Linux-only**. macOS and BSD have different module
systems and are not supported.

## License

GPL-3.0 — see
[LICENSE](https://github.com/orhun/kmon/blob/main/LICENSE).
Copyleft; redistributing modified versions requires releasing the
modifications under GPL-3.0 as well. Linking from your own GPL-
compatible scripts is fine.

## One Concrete Example

```bash
# 1. open the TUI (read-only as your user; sudo for load/unload)
sudo kmon

# inside kmon — useful keys (from the on-screen help, '?' to toggle):
#   ↑/↓ or j/k       move in module list
#   /               filter modules by name (e.g. 'nvidia')
#   l               load a module (prompts for name + params)
#   u               unload selected module
#   b               blacklist selected module (writes /etc/modprobe.d/)
#   r               show dependency / used-by tree for selected module
#   d               cycle dmesg follow on/off
#   c               copy selected module name to clipboard
#   ?               toggle help overlay
#   q               quit

# 2. pre-filter on launch and choose color theme
sudo kmon --color magenta --tickrate 250

# 3. accessibility / non-Unicode terminals
sudo kmon --unicode false

# 4. CI-friendly self-test (no TTY actions, just version check)
kmon --help
```

## Niche It Fills

**A discoverable surface for the Linux module subsystem.** The
underlying tools (`lsmod` / `modprobe` / `dmesg`) are correct but
unfriendly: `lsmod` is a flat unsorted dump, `modinfo` is one
module at a time, dependency direction is implicit, and
correlating "I just `modprobe`'d that" with the kernel-log output
means juggling two terminals. `kmon` collapses all of that into
one screen with arrow-key navigation, which makes kernel-module
work approachable for people who do it once a quarter (laptop
sleep weirdness, GPU driver swaps, USB serial adapter not coming
up) instead of as a daily job.

## Why use it

Three workflows where `kmon` is meaningfully faster than the raw
toolchain:

1. **"Why is `<feature>` not working" debugging.** Filter for the
   suspected module (`/nvidia`, `/btrfs`, `/iwlwifi`), see at a
   glance whether it is loaded, what its parameters are, who uses
   it, and what the kernel last said about it — no
   `lsmod | grep && modinfo && dmesg | tail` chain.
2. **Safe load/unload with immediate feedback.** When you `u` an
   in-use module `kmon` shows the dependency tree (so you know
   *why* it cannot be unloaded) instead of just printing
   `rmmod: ERROR: Module is in use`. When you `l` a module the
   bottom dmesg pane shows the load messages live, so a failure
   reason ("firmware not found", "unknown symbol") is visible
   without switching to another terminal.
3. **Blacklist drafting.** `b` writes to
   `/etc/modprobe.d/blacklist-kmon.conf` with the right syntax
   the first time, instead of you remembering whether the line
   is `blacklist foo` or `install foo /bin/false` for the kind
   of blacklisting you actually want.

For an LLM-CLI workflow on a Linux box, `kmon` itself is
interactive and not directly scriptable — but it is the right
*onboarding* tool for the human reviewer who has to approve a
suggested `modprobe.d` change the agent drafted. The agent
proposes the change in a PR; the human runs `sudo kmon` to
confirm the module exists, see its dependency tree, and verify
the blacklist syntax before merging.

## Vs Already Cataloged

- **Vs [`bottom`](../bottom/) / [`zenith`](../zenith/) /
  [`btop`](../btop/):** General-purpose system monitors —
  CPU, memory, network, processes — but they do not understand
  the kernel module subsystem at all. Use them for "is the
  machine healthy"; use `kmon` for "is this specific kernel
  module loaded with the right parameters and what does dmesg
  say about it".
- **Vs [`procs`](../procs/):** A modern `ps` replacement —
  process-level introspection. Orthogonal to `kmon`'s
  module-level focus. Both are the same kind of "Rust TUI
  wrapper around a venerable Unix tool" and ship together
  comfortably.
- **Vs raw `lsmod` + `modinfo` + `modprobe` + `dmesg -w`:**
  Same actions, four windows, no dependency tree visualization,
  no live correlation between an action and its kernel-log
  output. `kmon` is strictly a UX layer; nothing it does could
  not be done with the classic stack and tmux, but the discovery
  cost (which flag, which file, which directory) is much lower.
- **Vs `dkms` / `kmod` directly:** `dkms` rebuilds out-of-tree
  modules across kernel upgrades; `kmon` does not replace it.
  Use `dkms` to keep a third-party module compiling; use `kmon`
  to inspect / load / unload it.

## Caveats

- **Linux only.** No macOS, no FreeBSD, no WSL1. WSL2 with a
  custom kernel works in principle but most distro WSL2 kernels
  ship without module support enabled.
- **Root needed for write actions.** `kmon` will start as a
  regular user but `l` / `u` / `b` will fail without
  `CAP_SYS_MODULE`. Either run under `sudo` or restrict yourself
  to read-only inspection.
- **Blacklisting via `b` writes to
  `/etc/modprobe.d/blacklist-kmon.conf`.** That file is owned by
  `kmon` and overwritten on subsequent blacklist actions —
  do not hand-edit it. For permanent, hand-curated blacklists,
  keep your own `.conf` in `/etc/modprobe.d/` instead.
- **GPL-3.0 license** matters if you want to embed `kmon` in a
  closed-source appliance image; in that case stick to invoking
  it as a separate process (which is fine under GPL) rather than
  linking its crates.
- **TUI assumes a real terminal.** Rendering inside `script(1)`,
  CI logs, or a plain pager will be garbled. Use `--unicode
  false` for VT100-only terminals, but for headless automation
  call `lsmod` / `modprobe` directly instead.
