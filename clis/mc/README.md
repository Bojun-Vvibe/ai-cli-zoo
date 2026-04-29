# mc

- **Repo:** https://github.com/MidnightCommander/mc
- **Upstream homepage:** https://midnight-commander.org/
- **Version:** 4.8.33 (2025 cycle)
- **License:** GPL-3.0-or-later — [COPYING](https://github.com/MidnightCommander/mc/blob/master/COPYING) (SPDX: `GPL-3.0-or-later`)
- **Language:** C
- **Install:** `brew install midnight-commander` · `apt install mc` · `dnf install mc` · `pacman -S mc` · binary name is `mc`

## What it does

`mc` (Midnight Commander) is the venerable Norton-Commander-style two-pane
terminal file manager that has been the on-ramp to the Unix shell for three
decades and is still actively maintained out of the GNU project. Two
side-by-side directory panels, function-key hints across the bottom row
(`F3` view, `F4` edit, `F5` copy, `F6` move, `F7` mkdir, `F8` delete,
`F9` menus), a built-in text viewer + hex viewer + `mcedit` editor, and a
shell prompt under the panels that you can drop into without leaving the
TUI.

- Speaks **VFS** transparently — `cd` into a `.tar.gz`, a `.zip`, an `.iso`,
  an RPM/DEB package, an SFTP URL (`cd sh://user@host/path`), an FTP URL,
  or an `fish://` shell-over-ssh — same panels, same keybinds, no separate
  tooling.
- **User menu** (`F2`) is per-directory shell scriptlets you write once and
  the whole team triggers from a key, turning the file manager into an
  ad-hoc launcher / runbook surface.
- **Bulk rename + diff + tree view + find files** are all keyboard-driven
  and live in one binary, so a fresh box with `apt install mc` is a
  complete file-ops workstation over SSH with no per-tool config.
- **Subshell integration** keeps a real `bash` / `zsh` / `fish` running
  under the panels — `Ctrl+O` toggles the panels away to give you the full
  shell, `Ctrl+Enter` pastes the highlighted filename onto the command
  line, so the shell-versus-file-manager modal switch stays inside one
  process.

## Example

```sh
# Open in a directory
mc /var/log

# Open with two specific panels
mc ~/src ~/builds

# Browse a tarball as if it were a directory
# (inside mc) cd backup.tar.gz

# Browse a remote box over SSH (uses the host's sftp-server)
# (inside mc) cd sh://bojun@example.com/srv/data
```

## When to use it

Reach for `mc` when you are SSH'd into a server, need to copy / move /
diff / inspect files across two directories, and reaching for a
graphical SCP client or a dozen `cp` invocations is the wrong shape for
the work. Particularly strong on:

- Triage on a fresh server where the only thing installed is the distro
  default toolset — `mc` is in every major package repo, no Rust / Go
  toolchain dependency, runs over a 9600-baud serial console.
- "Pick files from `A`, drop them in `B`" workflows where the sequence
  is many small moves you want to confirm visually before each step.
- Working with archive files and remote filesystems through one uniform
  panel UI rather than learning each tool's flags.
- Teaching shell newcomers the file-system mental model — the visible
  two-pane layout makes "source vs destination" obvious in a way `cp`'s
  argument order does not.

Skip when you want a modern Rust file manager with async I/O, image
preview, and Lua plugins — pick [`yazi`](../yazi/), [`broot`](../broot/),
or [`xplr`](../xplr/) for that. `mc` is the reliable old-guard tool
that is on every box and never breaks.
