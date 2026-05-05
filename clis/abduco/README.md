# abduco

- **Repo:** https://github.com/martanne/abduco
- **Version:** v0.6 (latest tag, 2020-02-11; project is feature-complete)
- **License:** ISC ([LICENSE](https://github.com/martanne/abduco/blob/master/LICENSE))
- **Language:** C
- **Install:** `brew install abduco` · `apt install abduco` · `pacman -S abduco` · `pkg install abduco` · build from source with `make` (single C file, no external deps beyond libc + pty support) · binary name is `abduco`

## What it does

`abduco` provides **session attach/detach for terminal programs** —
the part of `tmux` that lets you start a long-running command, walk
away, close the SSH session, come back the next day, and reattach
to the same running process with its scrollback intact — and
nothing else. There is no window manager, no pane splitting, no
status bar, no scripting language, no config file. One binary, one
verb (`abduco -c name cmd...` to create, `abduco -a name` to attach,
`Ctrl-\` to detach), under 1000 lines of C, ISC-licensed, written
by the same author as the `vis` editor and the `dvtm` window
manager. The companion idiom is `abduco -c work dvtm` when you
*do* want windowing — `abduco` handles persistence, `dvtm` handles
layout, two small tools instead of one big one.

## When to pick it / when not to

Pick `abduco` when the only feature you need from `tmux` is
**"survive an SSH disconnect"**. Long-running build, training run,
deploy script, batch evaluation, pair-programming session, agentic
coding session that you want to keep running while you close the
laptop — start it under `abduco -c name cmd`, and you can
`abduco -a name` from any later shell to look at where it is. No
config, no learning curve, no `prefix` key conventions to teach a
new colleague. Detach is a single keystroke.

Skip it when you actually want **windows, panes, splits, or a
status bar** — that's [`tmux`](../tmuxp/) territory (or
[`zellij`](../zellij/) if you want a modern layout language).
Skip it when you want **session sharing across users with separate
cursors** ([`tmate`](../tmate/), [`upterm`](../upterm/),
[`sshx`](../sshx/)) — `abduco` is single-user attach, not multi-user
collaboration. Skip it when you want a TUI-driven session picker
([`sesh`](../sesh/)) — `abduco -l` lists sessions as plain text and
that's the entire UI.

## Why it belongs in the zoo

The zoo's terminal-multiplexer shelf is heavy on *combined* tools
that bundle persistence + windowing + scripting:
[`tmuxp`](../tmuxp/), [`tmuxinator`](../tmuxinator/),
[`zellij`](../zellij/), [`byobu`](../byobu/), [`mosh`](../mosh/),
[`dvtm`](../dvtm/), [`sesh`](../sesh/). `abduco` is the orthogonal
**unbundled** entry: just persistence, deliberately no windowing.
That makes it the right baseline reach for "wrap one long-running
command so I can detach", and it composes with `dvtm` (the same
author's pure-windowing companion) to recreate `tmux` from two
small ISC-licensed pieces. The ISC license and ~1000-line codebase
also make it the cleanest reach for embedded / busybox / minimal
container environments where shipping a full `tmux` is overkill.

## Example invocations

```bash
# Start a long-running build under a named session, detach later with Ctrl-\
abduco -c build make -j8 release

# List active sessions (one line per session: name, status, started)
abduco -l

# Reattach to the build session from any later shell (read-write)
abduco -a build

# Read-only attach (good for pairing or watching a CI session you don't own)
abduco -A build

# Pair with dvtm to get tmux-shaped persistence + windowing in two binaries
abduco -c work dvtm

# One-shot: run a script, persist its output, exit the session when it finishes
abduco -e ^q -c overnight-train ./train.sh --epochs 50
```
