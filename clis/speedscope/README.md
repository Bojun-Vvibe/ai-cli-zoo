# speedscope

> **An interactive web-based flame-graph viewer that opens
> CPU-profile JSON / `pprof` / Linux `perf` / Chrome trace /
> Firefox / Instruments / py-spy / stackcollapse / many other
> formats and displays them as a navigable flame graph
> (left-heavy, time-ordered, or sandwich view) directly in
> the browser** — no upload, no server, the page is the app.
> The CLI ships as `npx speedscope` / `npm i -g speedscope`.
> Pinned to **v1.25.0**
> ([LICENSE](https://github.com/jlfwong/speedscope/blob/main/LICENSE),
> MIT).

Source: <https://github.com/jlfwong/speedscope>

## TL;DR

`speedscope` is the standard "open this profile and let me
read it" viewer for the polyglot CPU-profile world. The format
list is the killer feature: it natively reads
`perf script` output, Linux `perf.data` (via `perf script`),
Brendan Gregg's `stackcollapse-*.pl` folded format, Chrome
DevTools Performance JSON, the Firefox profiler JSON, V8 /
Node.js `--prof` and `--cpu-prof`, the Instruments deep-copy
format from macOS, the `pprof` format from Go / `pprof` /
Python `pprof` exporters, the `pyspy` JSON, the `safari` /
`webkit` profile, the `chrome://tracing` dump, and `callgrind`
output — so a single binary covers most of the languages a
working engineer encounters in one week. The viewer offers
three navigation modes: **time-ordered** (the classic flame
chart, x-axis is time), **left-heavy** (samples grouped by
stack and sorted descending — the canonical "where did the
program spend its time" view), and **sandwich** (callees +
callers of a selected function — the inverse view useful for
"who is calling `malloc` so much"). The CLI ships the same
HTML/JS app: `speedscope profile.json` opens it in the local
default browser; the page is fully self-contained, so the
profile is never uploaded.

## Install

```bash
# zero-install — fetches latest, opens browser, exits when done
npx speedscope profile.json

# global install (pin a version in CI / dotfiles)
npm i -g speedscope@1.25.0
speedscope profile.json

# Homebrew (macOS / Linux)
brew install speedscope

# self-host the bundle (no Node required)
# download the dist tarball from the GitHub release and
# `python3 -m http.server` over it; or copy dist/release/* to
# a static-host bucket and bookmark it
curl -LO https://github.com/jlfwong/speedscope/releases/download/v1.25.0/speedscope-1.25.0.tgz
tar xf speedscope-1.25.0.tgz
python3 -m http.server -d package/dist/release 8000
# then drag profile.json onto http://localhost:8000
```

## What it actually is

A single-page web app that the CLI command launches in your
default browser:

1. **Universal profile reader.** The format auto-detector
   sniffs the file content (not the extension) and dispatches
   to the right importer. The supported list as of v1.25.0:
   - `pprof` (Go, Python `cProfile`-via-`pyperf`, others)
   - Chrome DevTools Performance JSON (`profile.cpuprofile`)
   - Firefox profiler JSON
   - Linux `perf script` text output
   - Brendan Gregg `stackcollapse` folded format
   - Apple Instruments deep-copy CSV
   - V8 `--prof` log + `--cpu-prof` JSON
   - Safari / WebKit profile
   - `chrome://tracing` JSON
   - `pyspy` JSON
   - Haskell GHC stack format
   - Callgrind / Cachegrind format
   - `papyrus`, `Tracy`, `Trace Event` formats
   - Many others — see the source `import/` directory
2. **Three views, one keystroke each.**
   - `1` time-ordered flame chart — x-axis is time, useful
     for "what happened during the slow request"
   - `2` left-heavy aggregated — samples grouped by stack
     identity, sorted descending; the canonical "where did
     time go" answer
   - `3` sandwich (callees / callers) — pick a function,
     see who calls it (top half) and what it calls (bottom
     half) with self / total time columns
3. **Local-first, zero-upload.** The viewer is a static
   bundle — no telemetry, no analytics, no server roundtrip.
   The profile bytes never leave the local machine. Suitable
   for analysing production-traffic profiles that contain
   sensitive function names / file paths without uploading
   to a SaaS profiler.

## When to choose

- **You have a profile in *some* format and want to read it
  quickly.** `speedscope` removes the "first install the
  language-specific viewer" step. A Go `pprof`, a Node
  `cpuprofile`, a `py-spy --format speedscope` dump, and a
  `perf script` text file all open the same way and render
  in the same UI, so the cognitive cost of switching
  languages is one keystroke.
- **You want to share a profile with a teammate without
  building infrastructure.** `speedscope` exports an
  embedded-data HTML file (one `.html` containing the bundle
  + the profile data inline) — Slack-attach it, the
  recipient opens it locally, no install required.
- **You want to inspect a profile without sending it to a
  third party.** Datadog Profiling, Pyroscope Cloud, Polar
  Signals, and Sentry Profiling all upload. `speedscope`
  runs locally — appropriate when the profile contains
  customer-data-shaped function names or proprietary code
  paths.

## Vs already cataloged

- **Vs [`py-spy`](../py-spy/) / [`samply`](../samply/):**
  complementary — those are *profilers* that *produce*
  profiles; `speedscope` is the *viewer*. The recommended
  pairing: `py-spy record -o profile.json --format
  speedscope -- python myapp.py` then `speedscope
  profile.json`. Same with `samply record ./binary` →
  `speedscope` opens the resulting `profile.json`.
- **Vs `pprof -http=:8080 profile.pb.gz` (Go's bundled
  viewer):** `pprof`'s built-in viewer is excellent for Go
  profiles but Go-shaped (graph view, source view tied to
  Go source paths, callgrind-style tree). `speedscope`'s
  flame-chart-first UI is closer to the Brendan Gregg /
  Chrome DevTools shape that most engineers find faster
  to read; both work — pick by which view you want first.
- **Vs Chrome DevTools Performance tab:** Chrome only opens
  Chrome-shape profiles. `speedscope` opens those *and*
  every other format, so it is the right tool when the
  profile didn't come from a Chrome page.
- **Vs Firefox Profiler (profiler.firefox.com):** Firefox
  Profiler uploads by default; `speedscope` runs locally.
  Both UIs are excellent — pick `speedscope` when local-
  only is required.

## Caveats

- **Single-file, no remote-fetch.** `speedscope` reads one
  profile per browser tab. If your profile is split across
  N processes (per-thread `perf` records, sharded
  distributed traces) you must concatenate / merge before
  opening — the [`pprof`](https://github.com/google/pprof)
  CLI's `merge` mode, or `stackcollapse`'s sum semantics,
  cover this.
- **Memory-bound on huge profiles.** A 500 MB
  `cpuprofile` will take seconds to parse and may exhaust
  the browser tab's heap. Down-sample at the producer
  side (`perf record -F 99` instead of `-F 999`,
  `py-spy --rate 100` instead of `1000`) or filter with
  `pprof -seconds` before opening.
- **No source-view of arbitrary languages.** Unlike Go
  `pprof -http`, `speedscope` does not display the source
  line for a sampled stack frame — it shows the function
  name and the file:line label only. For source-correlated
  reading, open the file in your editor at the line shown
  in the sandwich view.
- **Web app, not a TUI.** Despite being installable as a
  CLI, the actual UI is a browser tab — a graphical
  display is required. For pure-TUI flame-graph rendering
  use Brendan Gregg's `flamegraph.pl` (SVG, openable in
  any browser, but no interactive sandwich / left-heavy
  toggle).
- **Project pace is steady, not fast.** v1.25.0 is the
  current release; format-importer additions land
  occasionally. The core viewer is feature-complete and
  the maintainer treats it as a stable tool, not a
  product roadmap.
