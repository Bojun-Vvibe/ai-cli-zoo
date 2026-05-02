# hwatch

> **A modern `watch(1)` replacement that records every run,
> diff-highlights changes between intervals, and lets you
> scroll back through history** — single Rust binary, ANSI-aware,
> with a built-in TUI. Pinned to **v0.3.18**
> ([LICENSE](https://github.com/blacknon/hwatch/blob/master/LICENSE),
> MIT).

Source: <https://github.com/blacknon/hwatch>

## TL;DR

`watch` shows you the *current* output of a command on a
recurring interval and throws away everything in between.
`hwatch` keeps every snapshot in an in-memory ring (or a JSON
log file on disk), highlights the diff between adjacent runs
in real time, and gives you a TUI where `Tab` jumps between
the **history pane** (a scrollable list of every prior run with
its timestamp + exit code) and the **output pane** (where you
can toggle word-diff, line-diff, or "only changed lines"
modes). It also handles ANSI color codes correctly — so
`hwatch -c 'kubectl get pods'` doesn't render as a wall of
escape sequences — and can shell out properly via `--exec`
so pipelines (`hwatch -e 'ps aux | grep nginx'`) work without
quoting gymnastics. Bonus: `--logfile out.json` writes a
replayable JSON record of every run, which is the closest
thing the watch-family has to a black-box recorder.

## Install

```bash
# Homebrew (macOS / Linux)
brew install hwatch

# Cargo
cargo install --locked hwatch        # 0.3.18

# Arch Linux
pacman -S hwatch

# Verify
hwatch --version    # hwatch 0.3.18
```

## Example usage

```bash
# 1. Plain replacement for `watch -n 2 df -h` with diff highlighting on
hwatch -n 2 df -h
#    Inside the TUI:
#      Tab        switch focus between history list and output
#      d          cycle diff mode: none → watch (line) → line → word
#      p / n      previous / next snapshot in history
#      Space      pause / resume polling
#      h          toggle history pane
#      q          quit

# 2. Run a *shell pipeline* (watch(1) can't do this without sh -c)
hwatch -n 1 -e 'ps aux | grep -v grep | grep nginx'

# 3. Color-aware: keep ANSI from kubectl / ls --color
hwatch -c -n 5 -e 'kubectl get pods -A | grep -vE "Running|Completed"'

# 4. Record everything to a JSON logfile for postmortem
hwatch -n 1 --logfile incident-$(date +%s).json -e 'ss -tnp | head -50'
#    Replay later (history pane is fully scrollable on reopen)
hwatch -L incident-1714000000.json

# 5. Trigger a beep on first diff (handy for "wait until it changes")
hwatch -n 2 -B -e 'curl -s -o /dev/null -w "%{http_code}" https://example.com'

# 6. Run forever but exit when the command exits non-zero
hwatch -n 5 --exit-on-error -e 'health-probe.sh'
```

## Why this lives in the zoo

`watch(1)` is one of those Unix tools nobody ever upgrades
because nobody noticed it could be better. `hwatch` is the
upgrade: it adds the three things you always wanted —
**history**, **diffs**, and **ANSI-correct rendering** — without
changing the muscle memory (`-n`, `-c`, `-e`). The JSON log
mode quietly turns it into an incident-recording tool: leave
`hwatch --logfile foo.json -e ...` running on a bastion when
you start debugging, and you have a timestamped record of
every probe you ran without piping anything through `tee` or
`script`. That makes it the rare CLI that earns its slot in
both your interactive shell *and* your runbook.
