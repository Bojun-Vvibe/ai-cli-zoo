# somo

> **A human-friendly alternative to `netstat` / `ss` for socket
> and port monitoring** — a Rust CLI that lists open TCP/UDP
> sockets with PID, process name, address, port, state, and user
> in a colorized, sortable table, plus an interactive `--kill`
> mode that lets you arrow-pick a socket and SIGTERM the owning
> process from the same view. Pinned to **v1.3.3** (commit
> `fb752c9048c6710fea74b8a3a982da552a468c35`,
> [LICENSE](https://github.com/theopfr/somo/blob/main/LICENSE), MIT).

Source: <https://github.com/theopfr/somo>

## TL;DR

`somo` is what you reach for when you'd otherwise type
`lsof -i -P -n | grep LISTEN` for the hundredth time and squint
at the columns. It runs once, prints a colorized table sorted
by PID (or port, or state), and exits. The killer feature is
`--kill`: same table, but interactive — arrow keys pick a row,
Enter SIGTERMs the owning process, no second `kill -9 <pid>`
round-trip. Cross-platform (Linux + macOS), single static
binary, no daemon.

## Install

```bash
# Cargo (any OS with a Rust toolchain)
cargo install somo

# Homebrew (macOS / Linux)
brew install somo

# Arch Linux (AUR)
yay -S somo

# from a release binary (Linux x86_64 example)
curl -Lo somo.tar.gz "https://github.com/theopfr/somo/releases/download/v1.3.3/somo-x86_64-unknown-linux-gnu.tar.gz"
tar xf somo.tar.gz && sudo install somo /usr/local/bin/

# verify
somo --version    # somo 1.3.3
```

On Linux, listing other users' sockets needs `sudo` (or
`CAP_SYS_PTRACE`). On macOS, `sudo` is required to see PIDs for
sockets owned by other users.

## License

MIT — see [LICENSE](https://github.com/theopfr/somo/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. show all open TCP+UDP sockets, colorized table
somo

# 2. listening sockets only — the "what's bound to a port" question
somo --listen

# 3. filter by port (great for "who has 8080?")
somo --port 8080

# 4. filter by process name
somo --proc node

# 5. only IPv4, only TCP, only ESTABLISHED
somo --tcp --ipv4 --established

# 6. interactive kill mode — arrow-pick a row and Enter to SIGTERM
sudo somo --kill

# 7. JSON output for scripts (jq-friendly)
somo --json | jq '.[] | select(.state=="LISTEN") | .port'
```

## Niche It Fills

**Ergonomic socket inspection at the terminal.** `netstat` is
deprecated on most distros, `ss` is fast but cryptic
(`ss -tulpan` then squint), `lsof -i` is universal but slow and
verbose. `somo` is the colorized, sortable, filter-by-flag
middle ground — and the only one in this zone that ships an
interactive kill TUI in the same binary.

## Why use it

1. **Colorized, sortable, filterable in one flag pass.**
   `somo --listen --tcp --port 5432` is the kind of query that
   takes three pipes with `ss`/`lsof`. Output is a real table,
   not whitespace-aligned columns, so wide PIDs / long process
   names don't shift everything right.
2. **Interactive `--kill` mode.** Same table, arrow-navigable,
   Enter sends SIGTERM to the row's PID. Closes the
   "find-the-PID, copy, `kill <pid>`" loop that `lsof` / `ss`
   force on you.
3. **`--json` for scripts.** Stable schema, jq-friendly, so the
   same binary that gives you a pretty TUI also feeds a Loki /
   shell pipeline ("alert if anything other than `nginx` binds
   `:443`").

## Vs Already Cataloged

- **Vs [`killport`](../killport/):** killport is a one-shot
  "kill whatever owns this port" hammer — single command, no
  inspection. `somo --kill` is the inspection-first path: you
  see *all* sockets, sort and filter, then pick. Use `killport`
  in scripts ("free port 3000 before tests"), use `somo` when
  you don't yet know which port or process is the offender.
- **Vs [`procs`](../procs/):** orthogonal — `procs` is a
  modern process-table viewer (CPU, RSS, TTY) that does not
  enumerate sockets. Pair them: `procs` for "what's running,"
  `somo` for "what's bound."
- **Vs [`bandwhich`](../bandwhich/):** different question.
  `bandwhich` answers "which process is sending bytes right
  now?" (live packet capture). `somo` answers "which sockets
  are open and who owns them?" (snapshot of the kernel's socket
  table). Bandwhich needs raw-packet privileges; `somo` only
  needs ptrace/root for cross-user PID resolution.
- **Vs [`cyme`](../cyme/):** completely orthogonal — `cyme` is
  for USB topology, not network sockets.

## Caveats

- **Linux + macOS only.** No Windows port today; the kernel
  socket-table reader is platform-specific.
- **Cross-user PIDs need privileges.** Without `sudo` (Linux)
  or root (macOS), sockets owned by other users show up with
  `?` in the PID/process columns. Same constraint as `lsof`
  and `ss`.
- **Snapshot, not stream.** `somo` runs once and exits. There
  is no built-in `top`-style refresh loop — if you need that,
  wrap with `watch -n1 somo` or use `bandwhich` for live
  per-process bandwidth.
- **`--kill` sends SIGTERM, not SIGKILL.** Well-behaved
  processes will catch and clean up; stuck processes need a
  second pass with `kill -9 <pid>` (which `somo` deliberately
  does not do for you — by design).
