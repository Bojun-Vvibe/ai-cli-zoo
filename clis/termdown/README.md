# termdown

> **Countdown timer and stopwatch in your terminal, rendered as
> giant figlet digits** — a tiny Python CLI that takes a
> duration (`25m`, `1h30m`, `2026-12-31 23:59`) or runs as a
> stopwatch, draws the remaining / elapsed time as ASCII-art
> using `pyfiglet`, and on completion can blink, beep, or run
> an arbitrary shell command. Pinned to **v2.0.0**
> ([LICENSE](https://github.com/trehn/termdown/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/trehn/termdown>

## TL;DR

`termdown` is the answer to "I want a Pomodoro / meeting timer
that I can see from across the room without alt-tabbing". It
takes one argument — a duration string (`25m`, `1h`, `90s`,
`1h30m15s`) or an absolute time (`23:00`, `2026-12-31 17:00`)
— and fills your terminal with the remaining time in figlet
font, updating once per second. When it hits zero it can blink
the screen, ring the terminal bell, run a custom command, or
all three. Without an argument, it's a stopwatch that counts
up. The whole thing is one Python file plus `pyfiglet` and
`click`; install is `pipx install termdown`.

## Install

```bash
# pipx (recommended — isolated venv, command on PATH)
pipx install termdown

# pip (user-site)
pip install --user termdown

# Homebrew (macOS / Linux) — community formula via pipx
brew install pipx && pipx install termdown

# Arch (AUR)
yay -S termdown

# verify
termdown --version    # termdown, version 2.0.0
```

Pure-Python; runs anywhere CPython ≥ 3.8 runs (Linux, macOS,
WSL, BSD, Termux). No daemon, no config file, no state.

## Use it for

```bash
# 25-minute Pomodoro
termdown 25m

# Stopwatch
termdown

# Until an absolute time
termdown 17:00
termdown 2026-12-31 23:59

# Blink the terminal when done + ring the bell
termdown 5m --blink --critical 10

# Run a command on completion (notification, sound, anything)
termdown 25m --exec-cmd 'notify-send "Pomodoro done"; mpv ~/sounds/bell.ogg'

# Show the title above the digits
termdown 5m --title "Tea steeping"

# Pick a different figlet font
termdown 1m --font slant
termdown 1m --font doom

# Voice countdown for the last N seconds (uses `say` / `espeak`)
termdown 10s --voice say              # macOS
termdown 10s --voice espeak           # Linux

# Stopwatch with custom outdate-marker (mark a lap)
termdown
# press space to pause, lap, then enter to resume
```

Keys at runtime: `space` pause / resume, `r` reset, `l` lap (in
stopwatch mode), `+` / `-` add or subtract a minute, `q` quit.

## Why include it in a CLI catalog

1. **It's the canonical "big-digits in your terminal" timer.**
   Several variants exist (shell loops with `figlet`, `tty-clock`
   for a real clock, `peaclock` for a stylish minimal one), but
   `termdown` is the one that actually solves *both* directions
   (countdown to zero, stopwatch up from zero) with one CLI,
   accepts both relative (`25m`) and absolute (`17:00`)
   targets, and wires up a clean on-completion hook
   (`--exec-cmd`). For a Pomodoro / meeting-break / build-watch
   workflow that does not need a desktop notification daemon,
   it is the smallest viable tool.
2. **The `--exec-cmd` hook makes it composable.** A timer that
   ends and runs `notify-send` + plays a sound + posts to a
   webhook is just `termdown 25m --exec-cmd '<your stack>'`.
   That sidesteps the usual Pomodoro-app problem (each app
   reinvents notifications) by delegating to whatever you
   already use (`notify-send`, `terminal-notifier`,
   [`gum`](../gum/) `confirm`, an HTTP `curl` to your own bot).
3. **Pure Python, zero state.** No `~/.config/termdown/`, no
   sqlite, no background daemon — when the process exits the
   timer is gone. That makes it disposable: drop it in a
   one-shot tmux pane, in a `watchexec` chain, in an LLM
   agent's tool call (`termdown 30s --no-figlet`) without
   worrying about cleaning up state.

For an LLM-CLI workflow, `termdown 30s --no-figlet --quit-after 0`
gives an agent a foreground "wait 30 seconds, then exit zero"
primitive that is more readable in a tool transcript than
`sleep 30` (the elapsed time is visible in the captured stdout)
and that can be interrupted by Ctrl-C with a clean exit code.

## Vs Already Cataloged

- **Vs [`countdown`](../countdown/):** very close peer —
  `countdown` (antonmedv/countdown) is a Go binary doing the
  same job with a slimmer feature set (no stopwatch direction,
  no absolute-time target, no `--exec-cmd`). `countdown` wins
  on dependency footprint (single static Go binary, no Python
  runtime); `termdown` wins on features (stopwatch + countdown
  + absolute time + lap + on-finish command + voice) and on
  font selection (any `pyfiglet` font, not a fixed renderer).
  Pick `countdown` for "I just need a 5-minute timer", pick
  `termdown` for "this is my Pomodoro / meeting tool".
- **Vs [`peaclock`](../peaclock/):** orthogonal — `peaclock` is
  a *clock* (current time of day, fancy renderers, optional
  timer mode); `termdown` is a *timer* (count down to / up from
  a target). `peaclock` is what you put on the spare monitor;
  `termdown` is what you launch for one task and let exit.
- **Vs [`porsmo`](../porsmo/):** `porsmo` is a Pomodoro-cycle
  manager (work / short-break / long-break, configurable
  repetitions); `termdown` is a single-shot timer. If you want
  the ceremony of a full Pomodoro session manager, use
  `porsmo`. If you want "remind me in 25 minutes" without a
  state machine, use `termdown`.
- **Vs `sleep N && notify-send '...'`:** the shell one-liner
  has no visible progress and no way to pause / extend. The
  moment you want "5 more minutes" mid-timer, you reach for
  something with a TUI; `termdown +5` (well, send `+` keys at
  runtime) covers it.
- **Vs `tty-clock` / `tock` (not cataloged):** those are
  always-running clocks; not the same niche.

## Caveats

- **GPL-3.0, not MIT.** The licence is more restrictive than
  most catalog entries; a non-issue for end-user CLI use, but
  worth knowing if you plan to vendor the source into a
  derivative tool.
- **Python startup cost.** Cold-launch is ~150 ms (CPython
  import of `pyfiglet` + `click`); negligible for a timer you
  start once per Pomodoro, noticeable if you script
  `termdown 1s` in a tight loop. For sub-second precision you
  want a different tool.
- **Figlet rendering needs ~80 columns.** The default font
  needs a reasonably wide terminal; in a narrow split-pane
  (~40 cols) the digits wrap and look ugly. `--no-figlet`
  falls back to plain digits.
- **`--exec-cmd` runs once, then `termdown` exits.** It is not
  a recurring scheduler — for repeating reminders, wrap it in a
  shell loop (`while :; do termdown 25m; done`) or use
  [`porsmo`](../porsmo/).
- **`--voice` shells out.** The voice-countdown feature
  literally invokes `say` (macOS) or `espeak` (Linux); if
  neither is installed it silently no-ops. Not a bug, but the
  failure mode is "no voice", not "explicit error".
- **No persistence across termdown restarts.** If you `Ctrl-C`
  out, the elapsed / remaining time is lost. Use a logfile
  wrapper (`termdown 25m && echo "$(date) +25m" >> ~/.pomos`)
  if you want history; this tool will not do it for you.
