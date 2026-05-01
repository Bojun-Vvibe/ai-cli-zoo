# fswatch

> **A cross-platform file-change monitor that emits a stream of
> events on stdout** — a single C++ binary that abstracts over
> macOS FSEvents, Linux inotify, BSD kqueue, Solaris file events,
> and a polling fallback, so one shell pipeline `fswatch -o . | …`
> reacts to filesystem changes identically on every Unix.
> Pinned to **v1.20.1**
> ([COPYING](https://github.com/emcrisostomo/fswatch/blob/master/COPYING),
> GPL-3.0).

Source: <https://github.com/emcrisostomo/fswatch>

## TL;DR

`fswatch` is the answer to "I want to re-run command X every time
any file under directory Y changes" when the script you're going
to write would otherwise hardcode `inotifywait` (Linux-only),
`fsevents_watch` (Mac-only), or worst case a 1-second `sleep` loop.
Pointed at one or more paths, it watches recursively using the
best native API the host kernel exposes (FSEvents on Darwin,
inotify on Linux, kqueue on FreeBSD/OpenBSD, file-event-notification
on Solaris/illumos, polling everywhere else) and prints one line
per change to stdout. The output is by default the changed path;
`-x` adds the event flags (Created / Updated / Removed / Renamed
/ AttributeModified / etc), and `-o` collapses bursts of events
into a single `count` line per batch — which is the form you
usually want when piping into `xargs -n1 -I{} ./rebuild.sh`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install fswatch

# Linux package managers
# Arch:           pacman -S fswatch
# Debian/Ubuntu:  apt install fswatch
# Fedora:         dnf install fswatch
# Alpine:         apk add fswatch
# openSUSE:       zypper install fswatch
# Nix:            nix-env -iA nixpkgs.fswatch

# FreeBSD
pkg install fswatch-mon

# from source (autotools)
curl -LO "https://github.com/emcrisostomo/fswatch/releases/download/1.20.1/fswatch-1.20.1.tar.gz"
tar xzf fswatch-1.20.1.tar.gz
cd fswatch-1.20.1
./configure --prefix=/usr/local
make -j"$(nproc 2>/dev/null || sysctl -n hw.ncpu)"
sudo make install

# verify
fswatch --version    # fswatch 1.20.1 …
```

No daemon, no config file, no shell init. Pure CLI tool that
reads paths from argv (or `-` for stdin) and writes events to
stdout.

## License

GPL-3.0 — see
[COPYING](https://github.com/emcrisostomo/fswatch/blob/master/COPYING).
The library itself (`libfswatch`) is also dual-licensed
permissively for embedding; the CLI binary is GPL-3.0.

## One Concrete Example

```bash
# 1. minimal: re-run a build on every file change in $PWD
fswatch -o . | xargs -n1 -I{} make

# 2. one-line live test runner (-l 1 debounces 1 s of bursts)
fswatch -o -l 1 src tests | xargs -n1 -I{} pytest -x

# 3. event-flag-tagged output (-x), filter to just "Updated"
fswatch -x . | grep -v ' AttributeModified' | grep ' Updated'

# 4. exclude patterns (regex on full path); useful for editor swap files
fswatch \
  -e '\.git/' -e '__pycache__' -e '\.swp$' -e 'node_modules' \
  -o . | xargs -n1 -I{} ./run-checks.sh

# 5. emit just the changed file path (default), feed into a worker
fswatch . | while read -r f; do
  case "$f" in
    *.py)  ruff check "$f"   ;;
    *.go)  gofmt -w "$f"     ;;
    *.css) prettier -w "$f"  ;;
  esac
done

# 6. explicit monitor selection (override autodetect)
fswatch -m fsevents_monitor       .   # macOS
fswatch -m inotify_monitor        .   # Linux
fswatch -m kqueue_monitor         .   # *BSD
fswatch -m poll_monitor -l 0.5    .   # fallback, 500 ms poll

# 7. JSON-style output via custom format string (-f)
fswatch -x --format='{"path":"%p","flags":"%f","ts":%t}' .

# 8. monitor multiple non-contiguous paths
fswatch -o ~/code/proj-a/src ~/code/proj-b/src /etc/myapp |
  xargs -n1 -I{} systemctl --user reload myapp.service
```

## Niche It Fills

**Cross-platform "react to filesystem changes" without the
per-OS API.** Every Unix has a kernel-level file-event API, but
they are mutually incompatible: `inotify` (Linux) is per-watch-
descriptor and does not recurse; FSEvents (macOS) is path-
recursive and emits coalesced batches; kqueue (BSD) is per-FD
and exhausts your FD limit on big trees; Solaris has its own.
Every "live reload" tool either picks one OS and ignores the
others, or rolls its own abstraction layer. `fswatch` is that
abstraction layer extracted as a standalone binary (and a
`libfswatch` if you want to link it into your own program), so
shell scripts, Makefiles, and one-off pipelines get the same
behavior on a Mac laptop and a Linux CI box.

## Why use it

Three things `fswatch` does that picking one native API does not,
that explain why it survives in the era of `entr`/`watchexec`:

1. **Genuinely cross-platform with native backends.** It is not a
   polling-only tool that "works everywhere" by being slow — on
   each OS it dispatches to the kernel's real file-event
   mechanism, then falls back to polling only when no native
   backend exists or you explicitly ask for `-m poll_monitor`.
   The same `fswatch -o .` pipeline that takes <5 ms on Linux
   takes <5 ms on macOS, with no script-level OS detection.
2. **Plain stdout interface, composes with anything.** The output
   is one path per line (or one event-flag-tagged line with `-x`,
   or one custom-format line with `-f '…%p…%f…'`). That means it
   composes with `xargs`, `while read`, `awk`, `grep`, `parallel`,
   or any pipeline element you already know — no DSL, no config
   file, no `-c "shell command"` flag to escape-quote.
3. **Library + CLI split.** `libfswatch` is a stable C++ API
   (with C and Ruby/Python bindings via the same project) that
   GUI apps and language runtimes embed; the `fswatch` binary is
   a thin wrapper. So the same abstraction layer your bash
   pipeline uses is also under `mosh`, several editors, and
   build tools — fewer bugs from divergent implementations.

For an LLM-CLI workflow, `fswatch -o <project> | head -1` is a
synchronous "wait for any file to change in this tree, then
return" primitive that an agent can use to gate the next tool
call on a build / test / format step having actually happened,
without polling and without choosing a backend.

## Vs Already Cataloged

- **Vs [`entr`](../entr/):** `entr` is the closest peer and the
  spiritual successor. It is BSD-licensed, smaller, and has a
  built-in command-runner: `ls src/*.py | entr -r pytest -x`
  watches the listed files and re-runs `pytest`. Tradeoffs:
  `entr` reads the watch list from stdin (so you re-list with
  `find … | entr` to pick up new files), `fswatch` watches whole
  directories recursively without re-listing. Pick `entr` for
  the small, focused "watch this list, run this command" loop;
  pick `fswatch` when you want raw event stream to pipe into
  arbitrary downstream tools or when you need recursive
  directory watching with a fixed argv.
- **Vs [`watchexec`](../watchexec/):** `watchexec` is the modern
  Rust take that bundles the watcher + the command runner +
  exclusion rules + signal handling (`SIGTERM` previous run on
  new event). It is what most "re-run on save" workflows reach
  for in 2024+. `fswatch` stays as the Unix-pipeline primitive:
  emit events, let the user wire them. Pick `watchexec` for
  "re-run my command, manage process lifecycle for me"; pick
  `fswatch` when you want raw events to feed into something more
  complex than "re-run command".
- **Vs [`watchman`](../watchman/):** Watchman is Facebook's
  long-running daemon optimized for huge monorepos — it
  maintains an in-memory crawl of the tree, supports queries
  ("what changed since clock $C"), and is the engine under
  Mercurial / Buck / Jest's `--watch`. `fswatch` is a one-shot
  CLI with no persistent state. Pick Watchman for repos with
  >100k files and many concurrent consumers; pick `fswatch` for
  scripts and small-to-medium projects where a daemon is
  overkill.
- **Vs `inotifywait` (inotify-tools, not cataloged):** Linux-only
  peer. Same idea (emit events to stdout) but no macOS / BSD
  support, and recursion (`-r`) is implemented as a manual
  per-directory watch list, so very large trees hit the
  `max_user_watches` sysctl. `fswatch` papers over this.

## Caveats

- **GPL-3.0 license on the binary.** Embedding the binary
  in a redistributed proprietary product is restricted; the
  `libfswatch` library is dual-licensed for embedding but the
  `fswatch` CLI is not. For pure local / build-time / CI use
  this is irrelevant.
- **Event coalescing differs by backend.** FSEvents on macOS
  batches events with up to ~30 ms latency by design; inotify
  on Linux is near-real-time; the polling fallback is whatever
  `-l` says. Don't write tests that assert per-event ordering
  across platforms — use `-o` (count-only) when you just want
  "something changed" semantics.
- **macOS recursive watch can miss rapid renames.** A known
  FSEvents quirk: if a directory is renamed, then immediately
  renamed back, both events may collapse into one with the new
  flags. Critical workflows (e.g. atomic config swaps) should
  not depend on observing both halves.
- **No per-event command runner.** `fswatch` itself does not
  spawn the rebuild — you pipe to `xargs` / `while read` / a
  shell loop. That is by design (composability) but means
  process-management concerns (kill the previous run, debounce
  bursts, propagate signals) are your problem. Reach for
  `watchexec` if you want those bundled.
- **Filter expressions are POSIX regex on full path.** `-e
  'node_modules'` excludes anything whose full path contains
  the substring; `-i` includes. There is no `.gitignore`-style
  glob support — large monorepos with deep ignore rules need
  multiple `-e` flags or a wrapper that translates from
  `.gitignore`.
- **Watch limits on Linux still apply.** `fswatch -m
  inotify_monitor` is subject to
  `/proc/sys/fs/inotify/max_user_watches` (often 8192 or 65536
  default); a Node-modules-heavy tree blows past this. Either
  raise the sysctl (`sudo sysctl fs.inotify.max_user_watches=524288`)
  or exclude the noisy directories with `-e`.
