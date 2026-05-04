# thokr

> **Sleek typing TUI with visualised results and historical
> logging** — a small Rust binary that drops you into a typing
> drill in your terminal: a passage of words is rendered on
> screen, characters you have typed correctly turn green,
> mistakes turn red and stay marked, the cursor advances under
> the next character, and on completion you get WPM (gross +
> net), accuracy %, time elapsed, and a per-word error breakdown
> on a single results screen — pinned to **v0.4.1** (commit tag
> `v0.4.1`,
> [LICENSE](https://github.com/coloradocolby/thokr/blob/main/LICENSE),
> MIT).

Source: <https://github.com/coloradocolby/thokr>

## TL;DR

`thokr` is the modern Rust answer to the old `gtypist` /
`typespeed` lineage. It opens a `tui-rs` / `crossterm` full-
screen view, draws the prompt line, tracks every keystroke
(including backspace), and writes a per-session log line to
`~/.config/thokr/log.csv` (`timestamp,wpm,accuracy,seconds,
words_typed`) so a week of practice is a `cat log.csv | xsv
stats` away. Defaults to a 50-word random sample from the EFF
top-1000 wordlist; flags swap that for a fixed word count,
fixed time limit, or your own corpus piped in.

The killer property is the **deterministic results panel**: any
typing app gives you a number, but `thokr` shows the same number
on a stable layout (header, prompt area, hint line, footer) so
two screenshots a month apart are visually directly comparable —
the boring property that makes a practice habit stick.

## Install

```bash
# Cargo (works anywhere with a Rust toolchain)
cargo install thokr

# Homebrew (macOS / Linux)
brew install thokr

# Arch Linux (AUR)
yay -S thokr
```

Single static binary, ~3 MB, no runtime dependencies beyond a
modern terminal that supports ANSI + truecolour (every terminal
shipped in the last decade qualifies).

## Example usage

```bash
# default: 50 random words from the EFF top-1000
thokr

# 100-word run
thokr -w 100

# time-bounded: 60 seconds, words drawn until time runs out
thokr -s 60

# fixed prompt: drill the same passage repeatedly to track
# progress on a stable input
thokr -p "the quick brown fox jumps over the lazy dog"

# pipe in your own corpus (one prompt per invocation)
echo "deliberate practice beats casual repetition" | thokr -p -
```

Hot keys inside the run: any printable key types it; `Backspace`
deletes; `Esc` aborts; on the results screen `r` retries with
the same prompt, `n` draws a new prompt, `Esc` / `q` exits.

## Why it matters

Typing speed is the unglamorous floor under everything a CLI-
heavy operator does — pair-debugging, live shell sessions,
prompt iteration, code review on a flight. The honest measurement
of "am I getting faster or slower this quarter" needs (a) a
stable test harness, (b) a log you can plot, (c) zero friction
to launch. `thokr` is exactly that: one binary, one command, one
CSV. Pairs naturally with [`ttyper`](../ttyper/) (sibling Rust
typing tutor with a different word source and richer in-session
animation — pick `ttyper` for the prettier session UI, `thokr`
for the deterministic log + results screen) and with
[`smassh`](../smassh/) (Python / Textual typing trainer with
multiple "tests" modes — pick `smassh` if you want monkeytype-
style theme + mode variety, `thokr` for minimal Rust footprint).

## License

MIT. See
[LICENSE](https://github.com/coloradocolby/thokr/blob/main/LICENSE)
in upstream.

## As of

2026-05-04. Upstream tag `v0.4.1` (latest on
`coloradocolby/thokr` as of this snapshot). Versions and
dictionaries drift; re-check before pinning in CI.
