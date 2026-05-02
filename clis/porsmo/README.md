# porsmo

> **A pomodoro / timer / stopwatch TUI in Rust** — one
> ~2 MB binary with three subcommands (`pomodoro`, `timer`,
> `stopwatch`), big ASCII-rendered digits, and a unified
> duration syntax (`25m`, `1h25m3s`). Pinned to **v0.3.0**
> ([LICENSE](https://github.com/ColorCookie-dev/porsmo/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ColorCookie-dev/porsmo>

## TL;DR

`porsmo` is what `tomato` / `pomodoro` / `gone` would look
like if rewritten as one Rust binary that does the three
adjacent things you actually want from a timer: a classic
Pomodoro cycle (work / short break / long break with
configurable lengths), a one-shot countdown (`porsmo timer
45m`), and a stopwatch that survives terminal resizes. The
display uses big-block ASCII digits so it reads from the far
side of a desk; pause / resume / skip-phase are single keys
(`Space`, `S`); on completion the binary plays a short alert
beep through the system bell so a long-running build can yell
at you from a tmux pane you're not looking at. v0.3.0 dropped
the old colon-prefixed time syntax for the conventional
`1h25m3s` form, removed the implicit 25-minute default, and
collapsed the internal state machine — fewer surprises, same
footprint.

## Install

```bash
# Cargo (recommended — one binary, no runtime deps)
cargo install porsmo            # crates.io
# or pin to a tag from source
cargo install --locked --git https://github.com/ColorCookie-dev/porsmo --tag 0.3.0

# Arch Linux (AUR)
paru -S porsmo

# Verify
porsmo --version    # porsmo 0.3.0
```

## Example usage

```bash
# 1. Default Pomodoro cycle: 25m work / 5m short / 10m long,
#    4 work blocks before the long break
porsmo pomodoro

# 2. Custom Pomodoro: 50/10/30, 3 cycles before long
porsmo pomodoro -w 50m -s 10m -l 30m -n 3

# 3. One-shot 45-minute countdown (e.g. meeting timer)
porsmo timer 45m

# 4. Stopwatch — Space pauses, q quits and prints final time
porsmo stopwatch

# 5. Drive from a shell hook: ring after a build finishes
cargo build --release && porsmo timer 0s   # 0s = beep + exit
```

## Why this lives in the zoo

The pomodoro space is a graveyard of half-finished one-mode
binaries (just timer, just pomodoro, just stopwatch — never
all three). `porsmo` is the rare case where one ~2 MB Rust
binary covers all three with the same key bindings and the
same time syntax, so muscle memory transfers between modes
and the install step is one `cargo install`. It does not try
to be a productivity suite (no task list, no statistics, no
sync) — exactly the right scope for something you keep open
in a tmux pane next to your editor.
