# killport

> **Kill the process bound to a TCP/UDP port — by port number, not by
> PID.** A small Rust CLI that resolves "what's listening on `:3000`?"
> to a PID via the OS socket table, then sends `SIGTERM` (or any signal
> you pass) so the next `npm run dev` / `cargo run` / `uvicorn` no
> longer collides with a stale process from the last crashed terminal.
> Pinned to **v2.0.0**
> ([LICENSE](https://github.com/jkfran/killport/blob/main/LICENSE), MIT).

Source: <https://github.com/jkfran/killport>

## TL;DR

The dev-loop pain point: a server crashed without releasing its port,
or a `tmux` pane was killed without `Ctrl+C`, and now the next start
fails with `EADDRINUSE` / `address already in use`. The classic fix is
the three-step `lsof -i :3000` → eyeball the PID → `kill <PID>` ritual.
`killport 3000` collapses that into one command, with port-range
support (`killport 3000-3010`), explicit signal selection
(`killport -s SIGKILL 3000`), TCP/UDP filter (`-p tcp`), and dry-run
listing (`--dry-run`) so it doubles as a "who has my ports?" probe.

## Install

```bash
# Homebrew
brew install killport

# Cargo
cargo install killport

# Curl one-liner (vendor script)
curl -sL https://bit.ly/killport | sh

# Pre-built binaries on the releases page:
# https://github.com/jkfran/killport/releases/tag/v2.0.0

# Verify
killport --version    # killport 2.0.0
```

No elevated privileges required for ports owned by your user; killing
ports owned by other users (or system services) needs `sudo`.

## License

MIT — see [LICENSE](https://github.com/jkfran/killport/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. free up port 3000 (the canonical "EADDRINUSE on dev server" fix)
killport 3000

# 2. force-kill a stuck process (when SIGTERM is ignored)
killport -s SIGKILL 8080

# 3. range: free up everything in the typical dev-server band
killport 3000-3010

# 4. dry run — list what would be killed without sending a signal
killport --dry-run 5432 6379 8080

# 5. UDP only (e.g. a stuck DNS resolver test harness)
killport -p udp 5353

# 6. pre-flight a dev script
killport 3000 5173 8000 && npm run dev
```

## Niche It Fills

**Port-keyed process termination as a primitive.** Every dev knows the
`lsof | grep | awk | xargs kill` incantation; `killport` is that
incantation as a one-binary noun. The catalog already has process
viewers ([`procs`](../procs/), [`pik`](../pik/)) and process killers
that take PIDs, but no tool whose primary key is the port number — the
key the developer actually *has* in the error message.

## Why use it

1. **Port-as-key matches the failure mode.** `EADDRINUSE: :3000` gives
   you a port, not a PID. `killport 3000` is the inverse function with
   no intermediate translation step.
2. **Range + multi-port in one call.** `killport 3000-3010 5432 6379`
   is one syscall sweep across the dev stack — useful as a `predev`
   npm script or a `Makefile` `clean` target.
3. **Cross-platform same syntax.** Linux, macOS, Windows all take
   `killport <port>`. The underlying socket-table queries differ
   (`/proc/net/tcp` vs `lsof` vs `GetExtendedTcpTable`); the CLI
   surface does not.

## Vs Already Cataloged

- **Vs [`pik`](../pik/):** orthogonal — `pik` is an interactive
  fuzzy-finder process picker keyed by name / cmdline. `killport` is
  non-interactive and keyed by port. Use `pik` when you know the
  process name; `killport` when you only have the port.
- **Vs [`procs`](../procs/):** different verb — `procs` *views* (the
  modern `ps`); `killport` *acts*. They compose:
  `procs --tcp 3000` to inspect, `killport 3000` to terminate.
- **Vs `lsof -i :3000 | awk … | xargs kill` (not cataloged):** the
  same outcome in three commands and one `awk` invocation. `killport`
  is one command, no shell quoting, and works the same on macOS and
  Windows (where `lsof` is BSD-specific).
- **Vs `fuser -k 3000/tcp` (Linux util-linux):** closest peer.
  `killport` adds macOS + Windows, port ranges, and a friendlier
  default signal (TERM rather than KILL).

## Caveats

- **Best-effort PID resolution.** Sandboxed processes (macOS App
  Sandbox, some container runtimes) may hide the owning PID from
  user-space queries; `killport` will report "no process found" even
  when `lsof` run as root would have located it. Re-run under `sudo`
  when this happens.
- **TERM vs KILL semantics are yours to choose.** Default `SIGTERM`
  lets the process clean up (close DB connections, flush logs); pass
  `-s SIGKILL` only when the process ignores TERM.
- **Race on rebind.** Between `killport 3000` and `npm run dev`, a
  third process can grab the freed port. For dev loops this is a
  non-issue; for scripted automation prefer `killport 3000 && sleep
  0.2 && start_server`.
- **Not a service manager.** `killport` terminates; it does not
  restart. Pair with [`hivemind`](../hivemind/) /
  [`overmind`](../overmind/) / [`mprocs`](../mprocs/) for supervised
  dev-loop process trees.
