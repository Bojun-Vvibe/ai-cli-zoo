# s-tui

> **Terminal CPU monitor + built-in stress test** —
> live CPU frequency, utilization, temperature, power, and
> performance-governor graphs in a `urwid` TUI, with an
> integrated stress mode that loads every core so you can
> watch thermal throttling happen in real time.
> Pinned to **v1.4.0**
> ([LICENSE](https://github.com/amanusk/s-tui/blob/master/LICENSE),
> GPL-2.0).

Source: <https://github.com/amanusk/s-tui>

## TL;DR

`s-tui` ("stress terminal UI") reads `psutil` + `/sys/class/thermal`
+ `/sys/devices/system/cpu/cpufreq` and renders four scrolling
graphs side-by-side: **frequency**, **utilization**, **temperature**,
**power**. The bottom-right pane shows the current cpufreq
governor (and lets you change it interactively if you started
the tool as root). Press `s` to enter **stress** mode and the
built-in stresser pegs every logical core; the temperature
graph climbs, the frequency graph drops the moment the SoC hits
its thermal limit, and a `Throttling` banner lights up — turning
"is this laptop thermally healthy?" or "does this cooling mod
help?" into a 60-second visual experiment instead of a
spreadsheet. No daemon, no DB, output is the live TUI plus an
optional CSV log (`--csv`).

## Install

```bash
# pipx (recommended — isolated venv, single binary on PATH)
pipx install s-tui

# pip (user site)
pip install --user s-tui

# Homebrew (macOS / Linuxbrew)
brew install s-tui

# Debian / Ubuntu
sudo apt install s-tui

# Arch
sudo pacman -S s-tui

# Fedora
sudo dnf install s-tui

# verify
s-tui --version    # 1.4.0
```

## Examples

```bash
# Just watch — frequency / utilization / temperature / power
s-tui

# Watch + log every sample to CSV for later analysis
s-tui --csv --csv-file thermals.csv

# Skip the TUI, run the bundled stresser headless for 60s
s-tui -t --terminal-only       # text-mode, useful over flaky SSH
```

## Niche / category

System monitor — narrow slice. Not a general "htop replacement";
it does CPU thermal/frequency/power telemetry with a stresser
attached, and that is the point.

## When to use

- Diagnosing a thermal-throttling laptop or single-board computer
  ("does it actually hit its rated boost clock under sustained
  load, or does it cap at 1.8 GHz after 12 seconds?").
- Validating a cooling change (repaste, fan curve, undervolt) by
  running stress mode before and after and comparing the
  temperature plateau.
- Picking the right `cpufreq` governor for a workload — the
  sidebar shows the current governor and (as root) lets you
  flip between `performance` / `powersave` / `schedutil`
  while watching the frequency graph respond.

## When NOT to use

- You want a general process / memory / disk / network monitor —
  use [`bottom`](../bottom/) / [`btop`](../btop/) / [`htop`](../htop/);
  `s-tui` deliberately ignores everything that isn't CPU
  thermal/frequency/power.
- You're on macOS expecting per-core temperatures — Apple does
  not expose SMC sensors via `psutil`; `s-tui` will run but the
  temperature graph stays empty (use `macmon` / `asitop` instead
  on Apple Silicon).
- You need long-term historical trending — `s-tui` is a live
  monitor with at most a CSV side-output; ship the CSV into
  Prometheus / InfluxDB / Grafana for retention.

## Related

- [`bottom`](../bottom/), [`htop`](../htop/), [`gtop`](../gtop/) — general system monitors
- [`fastfetch`](../fastfetch/), [`macchina`](../macchina/) — one-shot hardware info
- `stress-ng` — the canonical CLI stresser (s-tui's built-in stresser is a thin reimplementation; use `stress-ng` for richer workload models)
