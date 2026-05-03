# termgraph

> **A small Python CLI that draws bar charts —
> regular, stacked, grouped, horizontal, vertical,
> and calendar / heatmap variants — directly in
> the terminal using Unicode block characters and
> ANSI 256-colour output**, taking the same kind of
> CSV / TSV / whitespace-separated input you'd
> hand to `awk` and rendering it as a chart you can
> read in a `tmux` pane, an SSH session, or a CI
> log. Pinned to **v0.7.6**
> ([LICENSE.txt](https://github.com/mkaz/termgraph/blob/main/LICENSE.txt),
> MIT).

Source: <https://github.com/mkaz/termgraph>

## TL;DR

You just ran `kubectl top pods --no-headers`,
`du -sh */`, `git shortlog -sn`, or
`SELECT day, count(*) FROM events GROUP BY day` —
and you got a column of numbers. The numbers are
fine for a script, but for a *human glance* you
want the visual ranking. `termgraph` is the
"throw it a two-column file, get a bar chart"
tool: feed it `name<TAB>value` (or pass `--delim ,`
for CSV), and it draws horizontal bars sized to
the data, optionally coloured, optionally
multi-series stacked, optionally as a calendar
heatmap. No matplotlib, no headless browser, no
PNG round-trip — pure stdout characters.

## Install

```bash
# pipx (recommended — isolates the env)
pipx install termgraph                    # latest
pipx install 'termgraph==0.7.6'           # pin

# pip
python3 -m pip install --user termgraph

# Verify
termgraph --help
```

`termgraph` is pure Python (≥ 3.8); the only hard
dependency is `colorama` for Windows ANSI support.
Drop into any container with Python and it works
— useful in CI logs that you'll later eyeball.

## Use it for

```bash
# 1) The headline use: two-column file → horizontal bar chart
cat <<EOF > sales.dat
2021: 184
2022: 209
2023: 411
2024: 502
2025: 388
EOF
termgraph sales.dat

# 2) Read from stdin (pipe-friendly)
du -sh */ | awk '{print $2, $1}' | termgraph

# 3) Multi-series grouped bars
cat <<EOF > by-team.dat
@ 2024,2025
backend  120,180
frontend  90,140
infra     60,80
EOF
termgraph --title "Tickets closed" by-team.dat

# 4) Stacked bars (series add up per category)
termgraph --stacked by-team.dat

# 5) Vertical (column) chart
termgraph --vertical sales.dat

# 6) Coloured bars (one colour per series)
termgraph --color {blue,red,green} by-team.dat

# 7) Calendar heatmap — one cell per day for a year
cat <<EOF > commits.cal
2025-01-01 3
2025-01-02 5
2025-01-03 0
... (one line per day)
EOF
termgraph --calendar --start-dt 2025-01-01 commits.cal

# 8) Custom delimiter (CSV)
termgraph --delim , sales.csv

# 9) Display values + percentage on each bar
termgraph --format '{:.0f}' --suffix '%' shares.dat
```

Long-form options control width
(`--width`), title (`--title`), the bar character
itself (`--custom-tick █`), and whether to print
labels above or beside the bars. Run
`termgraph --help` for the full enumeration.

## Why include it in a CLI catalog

1. **Charts where charts are normally banned.**
   CI build logs, cron-mailed reports, `tmux`
   panes during incident response, SSH sessions
   into a bastion that can't show images — all of
   them can carry Unicode and ANSI. `termgraph`
   gets you the *visual ranking* of values in
   exactly those environments. For "how lopsided
   is this distribution?" questions the bar chart
   is faster than reading 40 numbers.
2. **Same input shape as everything else in your
   pipeline.** Two columns, whitespace or custom
   delimiter — the exact output of
   `awk '{print $1, $N}'`, `cut -f1,2`,
   `sort | uniq -c | sort -rn | head`, or a SQL
   `GROUP BY ... ORDER BY count DESC`. No special
   schema, no pre-aggregation step beyond what
   you'd do anyway.
3. **Calendar heatmap built in.** GitHub-style
   contribution graphs (`--calendar`) are useful
   for any per-day count: deploy frequency, ticket
   closures, alert fires, exception bursts. Most
   chart tools make you assemble that visualisation
   yourself; `termgraph` ships it as one flag.

For an LLM-CLI workflow, `termgraph` is the
"render the model's tabular output as a glance-
able chart" piece: the model emits
`category<TAB>count`, you pipe through
`termgraph --color {blue}` and the operator sees
the shape immediately rather than having to read
JSON or import into a spreadsheet. Pairs naturally
with [`csvtk`](../csvtk/) /
[`miller`](../miller/) /
[`angle-grinder`](../angle-grinder/) on the
aggregation side.

## Vs Already Cataloged

- **Vs [`spark`-style sparklines](../spark/) (if
  cataloged) / inline bar tools:** different
  granularity. Sparklines fit on one line and lose
  the labels; `termgraph` uses multiple lines,
  keeps the labels, and supports multi-series and
  stacked layouts.
- **Vs [`gnuplot`](../gnuplot/) (terminal
  backend):** `gnuplot` is far more powerful —
  scatter, line, log scale, fits, contour — but
  also far more ceremony (script files, plot
  commands, terminal selection). For "I have two
  columns, give me bars", `termgraph` is one
  invocation.
- **Vs [`youplot`](../youplot/) /
  [`asciigraph`](../asciigraph/) / Python
  `plotille`:** orthogonal styles. `youplot` (Ruby)
  draws line / scatter / box plots in Unicode
  Braille; `asciigraph` (Go) is for time series
  line charts; `plotille` is a *library* you call
  from Python code. `termgraph` is the
  CLI-native, bar-chart-focused, calendar-aware
  member of the family — a script-friendly default.
- **Vs [`tickrs`](../tickrs/) /
  [`gping`](../gping/) / [`bandwhich`](../bandwhich/):**
  those are *live TUIs* for specific data sources
  (stocks, ping, network). `termgraph` is one-shot
  rendering of arbitrary tabular input.

## Caveats

- **Bar charts only.** No line, scatter, box,
  histogram, or pie. If the relationship you want
  to convey is a trend over time at sub-day
  resolution, reach for `asciigraph` or `youplot`
  instead.
- **Output renders in 24-row × 80-col-ish space
  by default.** Set `--width` for wider terminals.
  Very long category labels wrap awkwardly — pre-
  truncate with `awk '{print substr($1,1,18), $2}'`
  if you have 200-char identifiers.
- **Colour requires a terminal that speaks ANSI
  256-colour.** Fine for modern terminals,
  `tmux`, `screen`, GitHub Actions / GitLab CI
  logs. Strip with `--no-color` for log
  collectors that mangle escape codes.
- **No incremental / streaming mode.** Reads the
  whole input, then renders once. For tail-and-
  redraw use a TUI like [`tickrs`](../tickrs/) or
  [`gping`](../gping/).
- **MIT license** — permissive; safe to embed
  inside container images, vendor scripts, and
  CI helper layers with attribution.
