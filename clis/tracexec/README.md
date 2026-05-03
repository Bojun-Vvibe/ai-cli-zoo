# tracexec

> **A Linux process-tracer focused on `execve` / `execveat`
> — every time the kernel starts a new process under your
> command, `tracexec` prints the resolved binary path,
> argv, envp diff, fd table, cwd, and the search of
> `$PATH` that produced the match** — with a colored TUI
> mode (`tracexec tui --`) that gives you an interactive,
> pageable, filterable view of the process tree as it
> spawns. Pinned to **v0.17.0**
> ([LICENSE](https://github.com/kxxt/tracexec/blob/main/LICENSE),
> GPL-2.0).

Source: <https://github.com/kxxt/tracexec>

## TL;DR

`strace -f -e execve` answers "what did this thing exec"
but the output is dense, eats the terminal, and demands
post-processing to be useful. `tracexec` is the focused
modern tool for the same question: it traces only
`execve` / `execveat` (no read / write / mmap noise),
resolves each invocation to a fully expanded form
(absolute binary path, argv as a list, the env vars that
*differ* from the parent — not the full env every time),
and shows you why a particular binary was picked
(`PATH=...` walk, shebang resolution, interpreter chain).
It also ships a launcher mode for debuggers: `tracexec
gdb -- ./buildscript.sh` lets you intercept the *child*
process you actually care about and attach gdb at its
first instruction, instead of trying to attach to a
binary whose PID you do not know yet.

## Install

```bash
# Pre-built release (Linux x86_64)
curl -L https://github.com/kxxt/tracexec/releases/download/v0.17.0/tracexec-x86_64-unknown-linux-gnu.tar.gz \
    | tar xz
sudo install tracexec /usr/local/bin/

# Linux aarch64
curl -L https://github.com/kxxt/tracexec/releases/download/v0.17.0/tracexec-aarch64-unknown-linux-gnu.tar.gz \
    | tar xz
sudo install tracexec /usr/local/bin/

# Arch Linux (AUR)
paru -S tracexec

# Cargo
cargo install --locked --version 0.17.0 tracexec

# verify
tracexec --version    # tracexec 0.17.0
```

`tracexec` uses `ptrace(2)` and (optionally) seccomp
filtering. No special capabilities are required for
processes you can already trace as your own user; if
`/proc/sys/kernel/yama/ptrace_scope` is set to `2` or
higher, you will need `CAP_SYS_PTRACE` or root, exactly
like `strace`. Linux only — there is no macOS / Windows
equivalent because the underlying `ptrace` semantics do
not exist.

## Use it for

```bash
# Trace every exec under a build script — the headline use
tracexec log -- ./configure --enable-foo
# prints, for each exec:
#   pid 12345 cwd=/home/me/build
#   argv=["/usr/bin/cc", "-Werror", "-c", "main.c"]
#   filename=/usr/bin/cc
#   PATH search hit at /usr/bin/cc
#   env diff: +CFLAGS=-O2 +DESTDIR=/tmp/staging

# TUI mode: live, scrollable process tree
tracexec tui -- make -j8

# Limit depth (only direct children of the command, not deep grandkids)
tracexec log --max-depth=1 -- npm install

# Show only execs that resolved through a $PATH search (catch the wrong
# `python` from a venv-shim)
tracexec log --show-path-resolution -- ./run.sh

# Show only execs whose binary lives outside a whitelist of dirs (catch
# a build that secretly invokes /opt/something)
tracexec log --filter-cmdline-regex='^/opt/' -- ./build.sh

# JSON output for piping into a script / agent
tracexec log --output=json -- ./scripts/release.sh > execs.jsonl

# "Modify-then-exec" — pause at every exec and let you edit argv / env
# before the syscall actually completes. Surgical for debugging
# misbehaving wrappers.
tracexec collector --modify-argv -- ./flaky-wrapper

# Launch a child under gdb — gdb attaches at the *child's* entry point,
# not the parent shell's
tracexec gdb -- ./buildscript.sh
# (similarly: tracexec lldb -- …, tracexec strace -- …)

# Replay an exec with the exact captured argv + env (useful when a build
# step fails inside a wrapper that buries the real command)
tracexec collector --output-format=run-script -- ./build.sh > rerun.sh
chmod +x rerun.sh
./rerun.sh           # re-runs the failing exec with the same env

# Trace a long-lived daemon: attach to a running PID
tracexec log --pid=$(pgrep -n nginx)
```

The default text output is one block per exec,
deliberately formatted to be greppable: `pid=`,
`filename=`, `argv=` are stable prefixes. The TUI is the
right default when you want to *understand* what a build
is doing; `log --output=json` is the right default when
you want to *post-process* it.

## Why include it in a CLI catalog

1. **It is the focused modern answer to "what is this
   build script actually exec-ing".** This question comes
   up constantly: a Makefile that mysteriously picks the
   wrong compiler, a `pip install` that runs an
   unexpected shell hook, a CI job that pulls in a
   different `node` than the local dev env, an `npm
   run` that re-shells through three layers of wrappers.
   `strace -f -e execve` works, but the signal-to-noise
   ratio of `tracexec` (env *diff*, $PATH resolution
   trace, deduplicated wrapper chains) is dramatically
   better for this specific question.
2. **The launcher modes (`tracexec gdb -- ...`) solve a
   real debugger-attach problem.** When the binary you
   want to debug is spawned three layers deep inside a
   shell wrapper, attaching gdb manually requires
   guessing the PID before it exits. `tracexec gdb`
   ptraces the chain, identifies the target child by
   name / argv match, and hands gdb a stopped process at
   entry. Same trick for `lldb`, `strace`, `gdbserver`.
3. **First-class TUI and JSON outputs.** Most ptrace-
   adjacent tools (`strace`, `ltrace`, `bpftrace`) are
   stream-of-text. `tracexec tui` gives an interactive,
   filterable, expandable tree view; `tracexec log
   --output=json` gives one JSON object per exec with
   stable fields. Either makes "find the one bad exec
   in 5,000" tractable in a way that piping `strace`
   into `less` does not.

For an LLM-CLI workflow, `tracexec log --output=json --
./build.sh` produces a JSONL transcript an agent can
summarize ("the build invoked /usr/bin/python3 47 times
with PYTHONPATH overridden to /opt/legacy on three of
them") without parsing freeform `strace` text.

## Vs Already Cataloged

- **Vs [`strace`-style tools]:** closest peer in spirit
  is `strace -f -e execve`. `strace` is the universal
  syscall tracer; `tracexec` is the specialist for the
  exec subset, with much better presentation and
  launcher modes. Use `strace` when you need the full
  syscall surface (open, read, mmap, futex); use
  `tracexec` when you only care about which binaries
  ran and why.
- **Vs [`bingrep`](../bingrep/):** orthogonal —
  `bingrep` parses ELF / Mach-O / PE *files* statically.
  `tracexec` watches a *running* process tree. Pair
  them: `tracexec` finds the unexpected binary, `bingrep`
  tells you what is in it.
- **Vs [`hyperfine`](../hyperfine/):** orthogonal —
  `hyperfine` measures wall-clock time of a command;
  `tracexec` shows what that command does internally.
  Run `hyperfine` to learn that step X is slow, then
  `tracexec log -- step-X` to learn *why* (it forks
  500 sub-shells, or it execs a Python interpreter from
  a network mount, etc.).
- **Vs container debugging tools (`dive`, `lazydocker`):**
  orthogonal — those inspect the layered filesystem and
  declarative config of a container image. `tracexec`
  inspects what the container's *process* does at
  runtime. The combination ("this image is fine
  statically but something in the entrypoint execs an
  unexpected /opt/init binary") is a common discovery.

## Caveats

- **Linux only.** Built on `ptrace(2)` and Linux-specific
  process semantics. There is no macOS / BSD / Windows
  port and no plan for one — the underlying syscall
  surface is fundamentally different on those platforms.
  WSL2 works (it is a real Linux kernel).
- **`ptrace_scope=1` (the Ubuntu / Debian default)
  restricts cross-user tracing.** Tracing your own
  child processes (`tracexec log -- ./mycmd`) works
  unprivileged. Attaching to an existing PID owned by
  the same user usually works under `scope=1`. Anything
  cross-user, or when YAMA is set to `2`, needs root or
  `CAP_SYS_PTRACE`.
- **GPL-2.0 license.** Stricter than the MIT /
  Apache-2.0 norm in this catalog; for redistribution
  inside a proprietary product, read LICENSE. Pure end-
  user CLI / debugging use is unaffected.
- **`ptrace` overhead is non-trivial.** A traced
  workload runs measurably slower than untraced;
  expect 2–10× depending on exec frequency. Fine for
  diagnostics, wrong tool for "always-on production
  observability" (use eBPF-based tools like `bpftrace`
  or `parca` for that).
- **`--modify-argv` is a footgun.** Editing argv
  mid-syscall lets you redirect a build to a different
  compiler interactively, which is exactly as dangerous
  as it sounds. Treat it as a debugger feature, not a
  scripting feature.
- **Last release v0.17.0 (2026-03).** Active project
  with ~monthly releases through 2025–2026; the 0.x
  version line means the JSON schema and CLI flags
  occasionally evolve, so pin a specific version when
  scripting.
