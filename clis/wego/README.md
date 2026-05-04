# wego

> **Weather forecast in the terminal** with a colored ASCII
> art table — single static Go binary that pulls from any
> of eight weather backends (Open-Meteo, SMHI, OpenWeatherMap,
> WeatherAPI, Pirate Weather, Caiyun, WorldWeatherOnline,
> or a local JSON file) and renders a 1–7 day forecast in
> one of four frontends (`ascii-art-table`, `emoji`,
> `markdown`, `json`) with felt + measured temperature, wind
> speed and direction, viewing distance, precipitation
> amount and probability, and humidity — pinned to **2.4**
> ([LICENSE](https://github.com/schachmat/wego/blob/master/LICENSE),
> ISC).

Source: <https://github.com/schachmat/wego>

## TL;DR

`curl wttr.in` for people who want a real binary they
control. wego is the "I run my own client against my own
free API key" answer — pick a backend (Open-Meteo and SMHI
are key-less and free), put a default location in
`~/.config/wego/wegorc`, drop `wego` into your
`~/.zshrc` greeting or your tmux statusline, and you have a
self-hosted forecast that does not depend on a third-party
HTTP-rendered ASCII service staying up.

The killer property is **swappable backends and frontends
behind one config**. The same binary speaks to the free
Open-Meteo API today, switches to a paid Pirate Weather key
tomorrow when accuracy matters, and pipes its `json`
frontend through `jq` into a custom statusline tool the
day after — without changing anything but a config flag.

## Install

```bash
# Go install (most reliable — single command, any platform)
go install github.com/schachmat/wego@latest    # tracks master
# pin to the 2.4 tag explicitly for reproducibility:
go install github.com/schachmat/wego@v2.4

# Homebrew (macOS / Linux)
brew install wego

# Arch Linux (AUR)
yay -S wego

# Debian / Ubuntu
sudo apt install wego    # if available in your distro

# verify
wego --version
```

First run generates `~/.config/wego/wegorc` (or the
OS-equivalent `os.UserConfigDir()` location). Edit it to
pick a backend and add an API key if the backend needs one.

## Example usage

```bash
# default location, default forecast length
wego

# explicit location and day count (order does not matter)
wego London 4
wego 4 London
wego "New York, NY" 7
wego 60.17,24.94 3       # lat,lon also works

# emoji frontend (no monospace-font requirement)
wego --frontend emoji London

# JSON output for piping into a statusline / dashboard
wego --frontend json London | jq '.Forecast[0].Hourly[0].TempC'

# markdown table for embedding in a daily standup note
wego --frontend markdown London 3 >> standup-$(date +%F).md

# disable color (NO_COLOR also honored)
wego --monochrome
NO_COLOR=1 wego

# force a specific config file (handy for per-project setups)
WEGORC=./project-wegorc wego

# show the built-in man page
wego --man

# free, keyless backends — no signup, just edit wegorc:
#   backend=openmeteo  (global)
#   backend=smhi       (Sweden + neighbours)
```

A typical `~/.config/wego/wegorc` after first run + edits:

```ini
backend=openmeteo
frontend=ascii-art-table
location=London
days=3
units=metric
om-lang=en
```

## Why it matters

- **Self-hosted, no third-party rendering dependency.**
  `wttr.in` is wonderful but it is one HTTP service in
  Vienna. wego runs locally and talks to whichever
  upstream weather API the operator picks — no risk of
  the rendering server being down, rate-limited, or
  ASCII-mangled by a corporate proxy that strips ANSI.
- **Eight backends, swap with one config line.** The
  same binary reads from free key-less APIs (Open-Meteo,
  SMHI) for casual use and from paid high-accuracy APIs
  (Pirate Weather, OpenWeatherMap paid tier) for
  workflows where forecast quality matters. Switching
  is a one-line config edit, not a tool change.
- **Composable JSON frontend.** `wego --frontend json`
  emits structured forecast data; pipe through `jq`
  into [`gum`](../gum/) for a custom dashboard, into
  a tmux statusline, into a daily-standup markdown note,
  or into a Prometheus textfile collector. The `json`
  backend can also *consume* a local file, so test
  fixtures and offline replay are first-class.
- **Disk caching with TTL.** `--cache-ttl` caches API
  responses on disk to stay under free-tier rate limits
  even when invoked from every shell prompt. The
  upstream API gets one call per TTL window regardless
  of how many shells render the forecast.
- **Built-in man page and config docs.** `wego --man`
  prints a full POSIX man page with every flag and
  config key documented inline — no need to context-
  switch to a website to find out what `metric-ms`
  versus `si` does for unit display.
- **Stable, finished, low-maintenance.** wego has been
  in the niche for nearly a decade with a stable CLI
  and config-file shape. The 2.x line adds backends
  and frontends without breaking existing config files —
  exactly the property a long-lived dotfile dependency
  needs.

## Vs Already Cataloged

- **ai-cli-zoo does not currently catalog any terminal
  weather client.** wego is the first entry in this
  niche. It composes naturally with cataloged shell-
  prompt and statusline tools — drop `wego --frontend
  emoji` into a [`starship`](../starship/) custom command
  module, or pipe the `json` frontend into a custom tmux
  format string built with [`gum`](../gum/) helpers.
- **Vs `curl wttr.in`:** orthogonal on capability,
  different operational model. `wttr.in` is a hosted
  HTTP service that renders ASCII server-side — zero
  install, zero config, but one external dependency. wego
  is a local binary against an upstream weather API of
  your choice — slightly more setup (one config file,
  one optional API key) but no third-party rendering
  dependency, and the operator chooses the data source.
- **Vs the eight backend APIs directly (curl + jq):** a
  shell-script forecast against Open-Meteo with `curl`
  and `jq` is ~30 lines per backend per output format —
  wego unifies that into one binary that already knows
  the response shapes for eight providers and renders
  in four formats. Drop the boilerplate; keep the
  control over which provider and which renderer.
- **Vs phone weather apps / system widgets:** different
  surface entirely. wego earns its place when the
  workflow is keyboard-first — tmux statusline, terminal
  startup banner, CI dashboard, daily-standup markdown,
  ssh-into-server-and-check-local-weather — none of
  which a phone app helps with.
- **Vs heavyweight observability dashboards (Grafana
  with a weather data source):** Grafana is the right
  answer for fleet-wide aggregated weather over time
  for a real product (renewables forecasting, logistics
  routing). wego is the right answer for "what is the
  weather where I am right now and for the next three
  days, displayed in my terminal."

## License

ISC — see
[LICENSE](https://github.com/schachmat/wego/blob/master/LICENSE).
Permissive, OSI-approved, equivalent to a simplified BSD
license. Embed in personal dotfile repos, ship inside
internal admin tooling, vendor into a corporate terminal
banner — all unrestricted under ISC terms.

## Caveats

- **Most backends require an API key.** Only Open-Meteo
  and SMHI are free and keyless out of the box.
  WorldWeatherOnline no longer offers free keys at all
  (issue #83 upstream). For zero-friction setup, start
  with `backend=openmeteo`.
- **Default `ascii-art-table` frontend needs a UTF-8
  256-color terminal and a monospaced font with the
  weather glyphs.** DejaVu Sans Mono and most Nerd Fonts
  work; some minimal terminal fonts will render the
  ASCII art with mojibake. Fall back to
  `--frontend emoji` or `--frontend markdown` on
  constrained displays.
- **Forecast accuracy is the upstream backend's
  problem.** wego is purely the client; if Open-Meteo
  says 22 °C and the actual temperature is 17 °C, the
  fix is to switch backends, not file a wego bug.
- **Slow release cadence.** wego is a stable, mostly
  done tool — 2.4 (April 2026) is the latest tag and
  there are only five tagged releases historically. This
  is a feature for dotfile dependencies, not a bug; pin
  to `v2.4` and forget about it.
- **Locale handling is per-backend.** The `*-lang` config
  keys (e.g. `om-lang`, `owm-lang`) set the natural-
  language strings the backend returns; coverage varies
  by provider. Test before relying on a non-English
  language for production output.

## As of

2026-05-04. Upstream tag `2.4` (2026-04-11). Mature,
slow-cadence project with a stable CLI shape and config
file format across the 2.x line. Safe to pin to `2.4` in
shared dotfiles and onboarding scripts.
