# pik

> Snapshot date: 2026-05. Upstream: <https://github.com/jacek-kurlit/pik>

**Process Interactive Kill — a TUI process picker that signals what you
select.** `pik` (Process Interactive Kill) opens a fuzzy-searchable list
of running processes, lets you filter by command name / PID / port / path
/ user, preview the full command line and parent tree, and then send a
signal (`SIGTERM` by default, `SIGKILL` / `SIGHUP` / etc. on demand) to
one or many at once. It's the friendly answer to the "I need to kill that
zombie node process listening on 3000 but I forgot the PID" loop where
you'd otherwise reach for `lsof -i :3000` then `kill -9 <pid>`.

## Repo + version + license

- Repo: <https://github.com/jacek-kurlit/pik>
- Latest release: **`1.0.0`** (2026-04-25)
- HEAD on `main`: `7fd92c6`
- License: **MIT** —
  <https://github.com/jacek-kurlit/pik/blob/main/LICENSE>
- License path in repo: `LICENSE` (SPDX: `MIT`)
- Default branch: `main`
- Language: Rust (built on `ratatui` + `sysinfo` for cross-platform process enumeration)

## Install

```bash
# Cargo
cargo install pik

# Homebrew
brew install pik

# Or grab a release binary from
# https://github.com/jacek-kurlit/pik/releases/latest
```

```bash
# Launch the picker, type to fuzzy-filter, Enter to send SIGTERM
pik

# Pre-filter to processes whose command contains "node"
pik node

# Filter to whatever owns TCP port 3000
pik :3000

# Filter by user, by path prefix
pik !root
pik /usr/local/bin/

# Inside the TUI: F9 = SIGKILL, F1 = help, Space = multi-select
```

## Niche

The "**fuzzy process picker → signal**" slot. The classic toolkit is
`ps aux | grep foo`, copy a PID, `kill -9 <PID>`; or `pgrep foo`,
`pkill -9 foo` (and pray the regex doesn't match too much). `htop`,
[`btop`](../btop/), [`bottom`](../bottom/), and [`procs`](../procs/)
all let you kill from a list, but they're system-monitor-shaped: their
job is to *display* CPU / memory / I/O over time, with kill as a side
feature. `pik` flips the priority — the entire UI is the picker, the
preview pane shows the args / env / cwd you actually need to confirm
"yes, that's the one", and multi-select + signal-on-Enter make
"kill all dev servers" a single keystroke.

## Why it matters

- **Smart filter prefixes** — `pik` parses leading sigils so the same
  text box does what would otherwise be different commands: `:3000`
  resolves the port→PID lookup that needs `lsof` / `ss`, `!user` filters
  by owner the way `pgrep -u` does, `/path/` matches by executable path
  the way `pgrep -f` does. One mental model instead of three.
- **Multi-select + non-default signals** — `Space` toggles selection,
  `F9` sends `SIGKILL`, function keys cover `SIGHUP` / `SIGINT` /
  `SIGUSR1` etc. So "restart all my Postgres-related background jobs
  with `SIGHUP`" is a 2-keystroke operation instead of a `for pid in $(...)`
  loop.
- **Comparable CLIs** —
  [`htop`](../htop/) and [`btop`](../btop/) are system monitors that can
  kill; [`bottom`](../bottom/) and [`procs`](../procs/) modernise the
  display side; [`fzf`](../fzf/) + `ps -ef | fzf | awk '{print $2}' | xargs kill`
  is the DIY precursor; `killall` and `pkill` are the non-interactive
  baseline. Pick `pik` when the question is "*which* process do I want
  to kill" rather than "how is the system doing".
- **MIT, single static binary** — ~6 MB stripped, no daemon, no config
  required, works on Linux + macOS + Windows; `sysinfo` provides the
  cross-platform process enumeration so behavior is consistent across
  hosts.
