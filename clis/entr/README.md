# entr

> **Run an arbitrary command when files change** — a tiny C
> utility (~1 KLOC, BSD-portable) that reads filenames on stdin
> and re-runs a command whenever any of them is modified. Pinned
> to **v5.8** (latest tag,
> [LICENSE](https://github.com/eradman/entr/blob/master/LICENSE),
> ISC-style).

Source: <https://github.com/eradman/entr>

## TL;DR

`entr` is the smallest possible "watch this list of files, run
this command on change" primitive: one C binary, no config file,
no DSL, takes filenames on stdin and the command as positional
args. It pre-dates the modern `watchexec` / `nodemon` /
`cargo-watch` wave by a decade and is still preferred by people
who want zero install footprint and no language runtime
dependency. The interface — `find ... | entr <cmd>` — composes
with every tool that produces filenames.

## Install

```bash
# Homebrew (macOS / Linux)
brew install entr

# Linux package managers
# Debian/Ubuntu: apt install entr
# Arch:          pacman -S entr
# Fedora:        dnf install entr
# Alpine:        apk add entr
# Nix:           nix-env -iA nixpkgs.entr

# from source (POSIX C, ~30 seconds to build)
git clone https://github.com/eradman/entr && cd entr
./configure && make test && sudo make install

# verify
entr -v    # release: 5.8
```

## License

ISC-style (BSD-compatible) — see
[LICENSE](https://github.com/eradman/entr/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. re-run tests whenever any .py file changes
find . -name '*.py' | entr -c pytest -x

# 2. live-reload the dev server on save (-r restarts long-lived process)
ls *.go | entr -r go run main.go

# 3. clear screen between runs (-c) and pass the changed file (/_)
echo my.md | entr -c pandoc /_ -o my.html

# 4. watch a directory tree (entr does not auto-recurse — feed it the list)
find src -type f | entr -d make

# -d "directory" mode also exits when a NEW file appears in a watched dir
# (so a `while :; do find src -type f | entr -d make; done` loop picks up
# newly-created files; without -d new files are ignored until restart)

# 5. open the changed file in $EDITOR (interactive workflows)
ls *.txt | entr -p vi /_

# 6. compose with fd / rg for richer file selection
fd -e rs -e toml | entr -cs 'cargo check && cargo test'
```

## Niche It Fills

**The Unix-shaped file watcher: stdin in, command out, no
config.** Modern competitors (`watchexec`, `nodemon`,
`cargo-watch`) bundle filtering, debouncing, and ignore-rules
into the binary. `entr` does none of that — it reads the watch
list from stdin, so you select files with whatever tool you
already use (`find`, `fd`, `git ls-files`, `rg --files`). The
result is ~30 KB of C that does one job, has done it the same
way since 2012, and never breaks.

## Why use it

Three reasons to reach for `entr`:

1. **Composability with `find` / `fd` / `git ls-files`.** The
   watch list is just stdin. Want to watch only files tracked by
   git? `git ls-files | entr ...`. Want to exclude vendor
   dirs? `fd -E vendor | entr ...`. No bespoke ignore-rule DSL
   to learn — you already know the upstream tool.
2. **Tiny, ancient, stable.** ~1 KLOC of POSIX C, ISC-licensed,
   ships in every distro. No Rust toolchain, no Node, no
   Python, no Go runtime. Safe to install on a locked-down box
   where new dependencies are friction.
3. **Sensible defaults around long-lived processes.** `entr -r`
   sends `SIGTERM` (configurable with `-s`) and re-execs on
   change, so dev servers / `tail -F` / file-syncer loops "do
   the right thing" without you scripting the kill.

## Vs Already Cataloged

- **Vs [`watchexec`](../watchexec/):** the closest peer —
  `watchexec` is the modern Rust answer with built-in `.gitignore`
  awareness, debouncing, and recursive directory watching out of
  the box (no `find` pipe needed). Pick `watchexec` when "one
  binary, ignore-rules, recursion" is what you want; pick `entr`
  when "stdin-driven, ~30 KB, in every distro since forever" is
  what you want. Many sysadmins keep both: `entr` on servers
  where `apt install entr` is one keystroke, `watchexec` on dev
  laptops where the ergonomics shine.
- **Vs [`task`](../task/) / [`just`](../just/):** orthogonal —
  those are command runners (you ask for a target, they run it
  once). `entr` is an event loop on top of "run this command".
  Compose: `find . | entr -c just test`.
- **Vs `inotifywait` (Linux-only, not cataloged):** lower-level
  primitive — `inotifywait` reports raw filesystem events;
  `entr` is the loop layer that runs your command. `entr` works
  on macOS and BSD via `kqueue`; `inotifywait` is Linux-only.

## Caveats

- **No recursion built in.** `entr` watches exactly the files on
  stdin; new files in a directory are not picked up unless you
  restart, and `-d` only exits on new-file events (you re-loop
  externally). For "watch this whole tree forever" use
  `watchexec` or wrap `entr` in `while :; do ... | entr -d ...;
  done`.
- **Debounce is implicit, not configurable.** Bursts of saves
  collapse into one run, but the threshold is hard-coded
  (~milliseconds). Tools that fire many writes per save (some
  IDEs, `rsync`) can trigger redundant runs.
- **macOS file-descriptor limit bites at scale.** `entr` opens
  one fd per watched file; a `find . | entr` over a 100k-file
  tree blows past the default `ulimit -n 256` on macOS. Raise
  with `ulimit -n 10240` or watch a narrower set.
- **No JSON / structured output.** `entr` exists to run a
  command, not report events. If you need "tell me which file
  changed and what the event was" feed `inotifywait` /
  `fswatch` into a script instead.
- **`-r` `SIGTERM` may not stop ill-behaved children.** A
  process that ignores `SIGTERM` (or spawns its own children
  that do) can outlive the restart; pass `-s SIGKILL` if you
  need a guarantee.
