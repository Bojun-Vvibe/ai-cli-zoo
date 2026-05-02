# multitail

> **`tail -F` on steroids: many files in one terminal, in split
> windows, with per-window filters, colors, and merging.**
> A C TUI that turns a single terminal into a tiled / merged
> log-tailing dashboard with regex coloring and on-the-fly
> filtering.
> Pinned to **v7.1.5**
> ([LICENSE](https://github.com/folkertvanheusden/multitail/blob/master/LICENSE),
> GPL-2.0).

Source: <https://github.com/folkertvanheusden/multitail>

## TL;DR

`multitail` runs `tail -F`-equivalent on N files (or N command
outputs) and shows them in a single terminal session as either
**split panes** (each file in its own scrolling window) or a
**merged stream** (all files interleaved with per-source
prefixes / colors). Each window can apply independent regex
coloring, regex filtering, line conversion (e.g. timestamp
normalisation), and a separate scrollback buffer with
search. It can also tail the output of arbitrary commands
(`-l`), re-run a command on an interval (`-R`), and merge a
remote `ssh host tail` into a local pane. Single C binary,
ncurses-based, no daemon, no Python.

## Install

```bash
# Debian / Ubuntu
apt install multitail

# Fedora / RHEL (EPEL)
dnf install multitail

# Alpine
apk add multitail

# Arch (AUR)
yay -S multitail

# Homebrew (macOS / Linuxbrew)
brew install multitail

# from source
git clone https://github.com/folkertvanheusden/multitail
cd multitail && make && sudo make install

# verify
multitail -V        # MultiTail 7.1.5
```

## License

GPL-2.0 — see
[LICENSE](https://github.com/folkertvanheusden/multitail/blob/master/LICENSE).
Copyleft. Fine for use as a CLI tool on any system (running it
imposes nothing on your code or logs); the constraint matters
only if you intend to *bundle modified multitail source* into a
distributed product, in which case the derived work must also
ship under GPL-2.0. For pure operational use (sysadmin, on-call,
CI log inspection) the licence is invisible.

## One Concrete Example

```bash
# 1. Two log files side-by-side in horizontally-split panes
multitail /var/log/nginx/access.log /var/log/nginx/error.log
# top pane = access, bottom pane = error; each scrolls
# independently; b opens the per-pane scrollback search.

# 2. Four files in a 2x2 grid
multitail -s 2 app.log db.log redis.log queue.log
# -s 2 = 2 columns; multitail tiles into 2x2 because there are 4
# inputs.

# 3. Merge multiple files into one chronological stream with prefixes
multitail -i app.log -I db.log -I cache.log
# -i opens the first file in a window; -I MERGES the next file
# into the same window with a [filename] prefix per line — the
# canonical use case for "show me everything in time order".

# 4. Per-window regex coloring (highlight ERROR / WARN distinctly)
multitail -cS log4j /var/log/myapp.log
# -cS log4j applies the bundled "log4j" colorscheme: ERROR red,
# WARN yellow, INFO default, DEBUG dim. Custom schemes go in
# /etc/multitail.conf with `colorscheme:NAME` blocks.

# 5. Tail the output of a command, re-run every N seconds
multitail -R 5 -l "kubectl get pods -A"
# re-runs the command every 5s and shows the diff against the
# previous run highlighted — a poor-man's `watch` that also
# scrolls back through history instead of clobbering the screen.

# 6. Tail a remote file over SSH next to a local one
multitail /var/log/syslog -l "ssh prod-1 tail -F /var/log/syslog"
# left pane = local syslog, right pane = remote syslog over ssh;
# multitail handles the broken-pipe case by reconnecting on
# next interval.

# 7. Filter a noisy log to only show lines matching a regex
multitail -e "ERROR|FATAL" /var/log/myapp.log
# everything else is dropped before display; combine with -ev
# (inverse) to drop heartbeat lines.
```

## Niche It Fills

**The "watch many logs at once without booting an observability
stack" gap.** When you're SSH'd into a box at 2 AM and need to
correlate three log streams to figure out which subsystem
caused the cascade, the lightweight options are usually
`tmux` + three `tail -F` panes (no merging, no shared search,
no regex coloring) or `tail -F a b c` (interleaved by tail's
heuristic, no per-source labelling, no filtering). The
heavyweight options are Loki + Grafana / Splunk / Elastic
(requires a stack you do not have time to provision at 2 AM).
`multitail` is the operational middle: one binary, no daemon,
gives you tiled or merged tail with coloring / filtering /
search, runs on any Unix box you can SSH to.

## Why use it

Three things `multitail` does that the obvious alternatives don't:

1. **True merge with per-source prefixes.** `tail -F a b c`
   prints filename headers only when the active file *changes*;
   `multitail -i a -I b -I c` prefixes *every line* with its
   source, so a chronological merge stays parseable when
   sources interleave at high rate. This is the difference
   between "useless" and "actually correlatable" for the
   on-call use case.
2. **Per-window regex coloring + filtering, no config required.**
   `-cS log4j` (or any bundled colorscheme) gives you ERROR-red
   / WARN-yellow on the spot; `-e regex` filters the live
   stream. The closest grep-based equivalent is `tail -F | grep
   --color=always | less +F`, which loses paging-vs-tailing
   semantics and can't do per-window independent settings.
3. **Tails commands, not just files.** `-l "cmd"` and `-R N -l
   "cmd"` make any command (kubectl, journalctl with rare flags,
   a custom monitoring script) into a tailable source. So
   "watch pod state and the app log side by side" is one
   `multitail` invocation instead of `tmux` + `watch` + `tail`.

For an LLM-CLI workflow, `multitail` is the **operator-side
observation step**: an agent kicks off a long-running build /
deploy / migration that writes to several log files, the human
runs `multitail -i build.log -I deploy.log -I migrate.log` and
watches the merged stream with ERROR coloring while the agent
works. It's the thing you reach for when an agent's output is
"check the logs" and the logs are plural.

## Vs Already Cataloged

- **Vs [`lnav`](../lnav/):** `lnav` is the *log navigator* —
  understands a hundred log formats, indexes them, lets you SQL
  over fields, has a built-in pretty-printer. It is the right
  answer when you want to *analyse* a finite set of log files.
  `multitail` is the right answer when you want to *watch live*
  several streams at once with low ceremony. Different verbs:
  `lnav` = "explore", `multitail` = "monitor".
- **Vs `tail -F a b c`:** Plain `tail` interleaves output and
  prints filename headers only on file change, which is unusable
  at high rates. `multitail` either tiles into separate panes
  or merges with per-line prefixes. Use `tail` for one-file
  tailing in a script; use `multitail` for multi-file
  observation in a session.
- **Vs `tmux` + N `tail -F` panes:** `tmux` gives you panes for
  free, but no merging, no shared search, no per-pane regex
  coloring, no command-tailing. `multitail` gives you all of
  those without launching a multiplexer. Use `tmux` when you
  want one shell per pane; use `multitail` when you want one
  *log stream* per pane.
- **Vs `goaccess` / `journalctl -f`:** `goaccess` is a domain-
  specific (web access log) analyzer with HTML output;
  `journalctl -f` is systemd-only and one-stream. `multitail`
  is format-agnostic and multi-stream.

## Caveats

- **GPL-2.0, not MIT/Apache.** Running it imposes nothing on
  your software, but bundling modified `multitail` source into a
  distributed product means the bundled work must ship under
  GPL-2.0. For internal tools / sysadmin use this is invisible;
  for "embed multitail inside our SaaS appliance image" double-
  check the license downstream.
- **ncurses-only TUI.** No web UI, no JSON output, no metrics
  endpoint. If you want to forward "all ERROR lines from these
  three files" into Slack, you still need a separate pipeline.
- **Coloring rules are regex over text.** `multitail` does not
  parse structured log fields (JSON, logfmt). For structured
  logs the right answer is `lnav` or `jq -R 'fromjson?'` in
  front of multitail.
- **Tailing remote files is "ssh + tail in a subprocess".**
  When the SSH connection drops, the pane goes silent until
  multitail reconnects on the next interval. For high-uptime
  remote tailing reach for a real log shipper.
- **Tag cadence is slow but live.** v7.1.5 (2024) is the latest;
  the project has been actively maintained since 2003. Bug-fix
  cadence is "as needed", and the feature surface has been
  stable for years — which is what you want from a 2 AM tool.
