# procs

> **A modern replacement for `ps(1)` written in Rust** — single
> static binary that lists processes with coloured columns, human-
> readable units, TCP/UDP port mappings, Docker container names,
> tree view, regex / keyword search, and an interactive `top`-style
> watch mode. Pinned to **v0.14.11** (commit
> `f461cdb395c69730fd314b7d5d48de98f5bf1b42`,
> [LICENSE](https://github.com/dalance/procs/blob/master/LICENSE),
> MIT).

Source: <https://github.com/dalance/procs>

## TL;DR

`ps aux | grep foo | grep -v grep` is the meme; `procs foo` is the
fix. `procs` reads the same kernel process table as `ps` but
formats it for humans by default: columns are coloured by category
(PID / user / state / CPU / mem / start time / TTY / command),
sizes use `K`/`M`/`G` suffixes, the `Command` column wraps instead
of being truncated at 80 cols, and any positional argument is a
regex / substring filter applied across PID, user, command, and
arguments. The killer extras `ps` does not give you: a `--tree`
view that draws parent/child as ASCII branches, a `TcpPort` /
`UdpPort` column that maps each PID to the ports it is listening on
(no more `lsof -i` round-trip), a `Docker` column that resolves
container IDs to names, a `--watch-interval 1` mode that turns the
tool into a `top`-shaped live view, and a config-file system
(`~/.config/procs/config.toml`) for picking which columns appear in
which order. Cross-platform (Linux, macOS, FreeBSD, Windows).

## Install

```bash
# Homebrew (macOS / Linux)
brew install procs

# Cargo
cargo install --locked procs

# Linux package managers
# Arch: pacman -S procs
# Debian / Ubuntu: apt install procs        # 22.04+
# Fedora: dnf install procs
# Nix: nix-env -iA nixpkgs.procs

# Windows
# scoop install procs
# choco install procs
# winget install dalance.procs

# release tarball (any OS / arch)
curl -Lo procs.zip "https://github.com/dalance/procs/releases/download/v0.14.11/procs-v0.14.11-aarch64-mac.zip"
unzip procs.zip && sudo install procs /usr/local/bin/

# verify
procs --version    # procs 0.14.11
```

First run prints with default columns and a colour theme tuned for
dark terminals; pass `--theme light` (or set `PROCS_THEME=light`)
on a light background. No daemon, no setuid bit, no privileged
capabilities required for the read-only views — `Docker` and
`TcpPort` columns may need group membership (`docker`,
`/proc/net/tcp` perms) to fully resolve.

## License

MIT — see
[LICENSE](https://github.com/dalance/procs/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. show every process whose command, user, or args match "node"
procs node
# coloured columns: PID USER STATE CPU MEM VSZ RSS TTY START COMMAND

# 2. tree view, only your own processes
procs --tree --user "$(whoami)"

# 3. add the ports column and find what is listening on 5432
procs --insert TcpPort | rg ':5432'

# 4. live watch mode (top-style), refresh every 2 seconds, sorted by CPU
procs --watch-interval 2 --sortd CpuTime

# 5. show docker container name alongside PID for compose debugging
procs --insert Docker postgres

# 6. JSON output for a pipe (scripting / dashboards)
procs --json node | jq '.[] | {pid: .pid, rss: .rss, cmd: .command}'

# 7. only processes using more than 500 MB resident
procs --sortd RssBytes | head -20

# 8. pin a column layout in config and forget the flags
mkdir -p ~/.config/procs && cat > ~/.config/procs/config.toml <<'EOF'
[[columns]]
kind = "Pid"
[[columns]]
kind = "User"
[[columns]]
kind = "CpuTime"
[[columns]]
kind = "RssBytes"
[[columns]]
kind = "TcpPort"
[[columns]]
kind = "Docker"
[[columns]]
kind = "Command"
EOF
procs        # uses the layout above
```

## Niche It Fills

**The TTY-aware `ps` for humans.** When you are at a shell looking
for "the Postgres process eating memory" or "what is listening on
:5432", `ps` and `lsof` and `docker ps` each give you one slice;
`procs` gives you all three in one coloured table with substring
filtering as the positional argument. The default layout is shaped
for the question "what is going on, on this box, right now?" — not
for shell-script parsing (use `procs --json` or stick with `ps -o`
for that).

## Why use it

Three things `procs` does that `ps` does not, that pay back the
muscle-memory cost:

1. **Positional arg is a substring / regex filter, across every
   column.** `procs foo` matches `foo` in PID, user, state,
   command, and arguments — no more `ps aux | grep foo | grep -v
   grep`. Anchored regex works too: `procs '^postgres'`.
2. **`TcpPort` / `UdpPort` / `Docker` columns are first-class.**
   `procs --insert TcpPort` resolves listening sockets to PIDs in
   the same table — a single command answers "what is on :5432
   and how much memory is it using"; `procs --insert Docker`
   labels every PID inside a container with that container's
   name, which is the missing link between `docker ps` (which
   gives you container names) and `top` (which gives you PIDs).
3. **Coloured, wrapping, tree-view default.** Long command lines
   wrap instead of being truncated, columns are coloured by type
   so you can spot a runaway-CPU process at a glance, and
   `--tree` draws parent / child relationships as ASCII branches
   — which makes "what spawned this?" answerable without piping
   `ps -ef` to `awk`.

For an LLM-CLI workflow the JSON output (`procs --json`) is the
useful shape: pipe it into [`llm-jq`](../llm-jq/) /
[`llm`](../llm/) to ask "which of my processes is closest to the
RSS limit and what command spawned it" and get a structured answer
back.

## Vs Already Cataloged

- **Vs [`bottom`](../bottom/):** orthogonal — `bottom` is a
  full-screen `top`/`htop`-style dashboard (CPU graphs, network /
  disk meters, sortable process pane, batteries, sensors); `procs`
  is a one-shot table that prints and exits, optimised for
  "answer this question and give me the prompt back". Use
  `bottom` to *watch* a system for minutes; use `procs` to
  *query* it in seconds.
- **Vs `ps` / `top` / `htop` / `lsof` (POSIX / classic):** `ps` is
  the right answer in shell scripts (`-o` is portable, output is
  trivially parseable, no colour codes); `top` / `htop` are the
  right answer when you need an interactive sortable dashboard.
  `procs` is the right answer for the 80% middle: "what is
  running, who owns it, how much is it using, and is it listening
  on a port?" — one command, coloured output, substring filter,
  done. You keep `ps` for scripts and `htop` for sit-and-watch.
- **Vs [`dust`](../dust/) (`du` replacement) / [`fd`](../fd/) /
  [`ripgrep`](../ripgrep/) / [`bat`](../bat/) / [`eza`](../eza/):**
  same family of "Rust rewrite of a classic Unix tool with sane
  colour defaults and a humans-first layout" — `procs` is the
  process-table member of that set. Once you have one of them in
  your terminal muscle memory the rest carry the same ergonomics
  (`--json` output, regex positional, colour-by-default,
  TTY-aware).

## Caveats

- **Output is not a stable contract.** Column set, ordering, and
  colour codes are tuned for human reading and have changed
  between minor versions; do not parse the default text output in
  scripts — use `--json` (the JSON shape is the closest thing
  there is to a stable contract).
- **Some columns need privileges or group membership.** `Docker`
  needs read access to the docker socket (group `docker` on
  Linux); `TcpPort` / `UdpPort` need `/proc/net/{tcp,udp}` reads
  on Linux, which work for your own sockets but show `-` for
  other users' sockets unless run as root. macOS uses `lsof`
  under the hood for ports and is correspondingly slower.
- **Watch mode is not a `top` replacement.** `procs --watch-
  interval 1` re-renders the whole table each tick; it does not
  diff, does not have keyboard sort / kill / filter bindings, and
  does not paint smoothly on slow terminals. For a real
  dashboard reach for [`bottom`](../bottom/) or `htop`.
- **Windows column set is reduced.** No `Docker`, no `Tcp/UdpPort`
  by default on Windows (the underlying APIs are different); the
  process / CPU / memory columns work fine.
- **Config file format is TOML and not auto-generated.** First
  time you want a non-default layout you have to write
  `~/.config/procs/config.toml` by hand from the docs; there is
  no `procs --init-config` command.
- **Cold start parses `/proc` for every PID.** On a box with
  thousands of processes (build farm, Kubernetes node) a single
  `procs` run can take several hundred milliseconds — fine for
  interactive use, surprising in a tight loop. Filter early
  (`procs --pid 1234,5678`) when you can.
