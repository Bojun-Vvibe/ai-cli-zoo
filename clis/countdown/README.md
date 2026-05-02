# countdown

> **Single-binary terminal countdown timer that renders
> giant ASCII digits and chains into a follow-up shell
> command** — `countdown 25m && say "break over"` is the
> entire mental model, and `Space` pauses, `Esc` aborts
> without firing the next command. Pinned to **v1.5.0**
> ([LICENSE](https://github.com/antonmedv/countdown/blob/master/LICENSE),
> MIT).

Source: <https://github.com/antonmedv/countdown>

## TL;DR

`countdown` is a ~600-line Go TUI that does exactly one
thing: display a countdown (or count-up) timer in
oversized ASCII figlet-style digits centered in your
terminal, and then exit cleanly so the next command in
your shell pipeline runs. Duration accepts Go's standard
`time.ParseDuration` syntax (`25s`, `1h30m`, `2h15m30s`)
or an absolute clock target (`14:15`, `02:15pm`) — the
latter is the killer feature for "alert me when this
meeting actually ends" workflows where you don't want to
do mental math on remaining time. Three flags cover the
remaining surface area: `-up` flips it to a stopwatch
counting from zero, `-say 10s` triggers the macOS `say`
command for the last N seconds (audible final-countdown
without writing a wrapper script), and `-title "..."`
prints a label below the digits. That's it. Because the
whole tool exits with status 0 only when the timer
completes (not when you `Esc`), you can chain it with
`&&` for fire-and-forget reminders that respect early
abort: `countdown 5m && terminal-notifier -message "5
min done"`, `countdown 1h && pkill -USR1 focus-app`, or
build a real Pomodoro out of `countdown 25m && countdown
5m && say done` without pulling in a Pomodoro app.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install countdown

# Go (1.21+)
go install github.com/antonmedv/countdown@latest

# Single-binary download (GitHub releases, v1.5.0)
curl -L -o countdown.tar.gz \
  https://github.com/antonmedv/countdown/releases/download/v1.5.0/countdown_Linux_x86_64.tar.gz
tar xzf countdown.tar.gz && sudo mv countdown /usr/local/bin/

# macOS arm64
curl -L -o countdown.tar.gz \
  https://github.com/antonmedv/countdown/releases/download/v1.5.0/countdown_Darwin_arm64.tar.gz
tar xzf countdown.tar.gz && sudo mv countdown /usr/local/bin/

# Build from source pinned to release tag
git clone --depth 1 --branch v1.5.0 https://github.com/antonmedv/countdown.git
cd countdown && go build -o countdown .
```

## Usage

```bash
# Plain duration
countdown 25s
countdown 1h2m3s

# Target time (today; tomorrow if already past)
countdown 14:15
countdown 02:15pm

# Chain a follow-up command
countdown 1m30s && say "break over"
countdown 25m && terminal-notifier -title "Pomodoro" -message "Focus done"

# Stopwatch (count up from 0 until you Ctrl+C)
countdown -up 30s

# Audible last-10s warning (macOS)
countdown -say 10s 1m

# Labelled timer
countdown -title "Code review" 30m

# Pomodoro: 25m focus, 5m break
countdown -title "Focus" 25m && countdown -title "Break" 5m && say "Cycle complete"

# Meeting-end alarm at an absolute time
countdown 16:30 && say "wrap it up"
```

Key bindings while the timer runs:

- `Space` — pause / resume
- `Esc` or `Ctrl+C` — abort, **and skip the chained command**

That last detail is the contract that makes `&&` chains
safe: if you abort, downstream commands don't fire.

## Why it's interesting

The terminal-timer slot is full of one-off bash one-liners
(`sleep 1500 && say done`, `for i in {1500..1}; do echo
$i; sleep 1; done`) and heavyweight Pomodoro TUIs
([`porsmo`](../porsmo/), [`gone`](../gone/),
[`pomo`](../pomo/), the various tomato-themed apps) that
ship state machines, configs, and statistics. `countdown`
is the rare middle-ground: it does one thing, has zero
config, and composes with the shell instead of replacing
it. The composition story is what sets it apart — because
exit status reflects "did the timer actually complete",
you can build any workflow that "do X for N minutes" tools
provide, in plain shell, without buying into their
abstractions. Pick `countdown` when (a) you want
a visible-from-across-the-room timer in a terminal pane
during meetings or focus blocks, (b) you want the
absolute-time form (`countdown 14:15`) so you don't have
to compute "minutes until 2:15pm" yourself, or (c) you
want a primitive to compose into your own Pomodoro,
deploy-window, or backup-window scripts. Not the right
pick when you need persistent statistics across sessions
(use [`porsmo`](../porsmo/) or
[`timewarrior`](../timewarrior/)), when you want
multiple parallel named timers (use [`tdr`](../tdr/) or
[`focus`](../focus/) where applicable), or when you need
a daemon that survives terminal close (use `at`, `systemd
--user` timers, or `launchd`). Project is small and
stable — last release v1.5.0 in September 2023, MIT, and
the entire source fits in `main.go` + `ui.go` + `font.go`
which makes it a good reading exercise if you want to see
a minimal Bubble Tea-style TUI in Go (it predates Bubble
Tea and uses `tcell` directly). Compare with
[`peaclock`](../peaclock/) (full-screen clock + timer
with theming, much heavier UI surface) and
[`hwatch`](../hwatch/) (a `watch` replacement that runs a
command repeatedly — a different problem) —
`countdown` occupies the "single-purpose, shell-
composable, big-digit timer" slot they don't.
