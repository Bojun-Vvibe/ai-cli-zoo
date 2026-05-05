# youplot

> **Single-binary CLI that draws Unicode bar / histogram /
> scatter / line / box / density / heat-map plots directly in
> the terminal from CSV / TSV / JSON on stdin** — wraps the
> Ruby `UnicodePlot` library so a `cat data.csv | uplot bar
> -H` is the entire pipeline from raw column to publishable
> ASCII chart, with no GUI, no DISPLAY, no `matplotlib`,
> no PNG round-trip. Pinned to **v0.4.6** (released
> 2023-03-10,
> [LICENSE.txt](https://github.com/red-data-tools/YouPlot/blob/v0.4.6/LICENSE.txt),
> MIT).

Source: <https://github.com/red-data-tools/YouPlot>

## TL;DR

The terminal-plotting CLI space splits into (a) static-image
emitters that write PNG / SVG to a file you then have to
`open` — `gnuplot`, `plotters-cli` — useless over SSH or in
CI logs; (b) sixel / kitty-graphics renderers — beautiful but
require a graphical terminal and break in `tmux`, `screen`,
and most CI viewers; and (c) pure-Unicode plotters that
target the lowest-common-denominator 80x24 character grid
that *every* terminal supports. `youplot` (binary name
`uplot`) is the most complete entry in category (c): seven
plot types (bar, histogram, line, scatter, density, box,
count, heat-map), CSV / TSV / JSON / space-delimited input,
column-by-name addressing (`-H` for header row), grouping
and stacking, log axes, custom colors per series, and a
streaming mode (`uplot bar -s`) that redraws the chart as
new lines arrive on stdin — turning `tail -f metrics.csv`
into a live dashboard inside any terminal.

## Install

```bash
# macOS (Homebrew)
brew install youplot

# Ruby (any platform with Ruby 2.7+)
gem install youplot

# Arch Linux (AUR)
yay -S youplot

# Verify
uplot --version           # YouPlot v0.4.6
```

## License

MIT — see
[LICENSE.txt](https://github.com/red-data-tools/YouPlot/blob/v0.4.6/LICENSE.txt).
Permissive: bundle into proprietary tooling, redistribute
modified, sell — only the copyright + permission notice must
travel with the source. The runtime is Ruby (MRI 2.7+); no
Python / Node / native graphics dependency, so it installs
cleanly into a slim CI image with `apk add ruby` or `apt-get
install -y ruby` plus `gem install youplot` and nothing else.

## Common invocations

```bash
# Bar chart from a 2-column CSV with header row
cat sales.csv | uplot bar -H -d,

# Histogram of a single numeric column
cut -f3 measurements.tsv | uplot hist --nbins 30

# Scatter of columns 1 vs 2, grouped by column 3
uplot scatter -H -d, --canvas dot --color blue data.csv

# Line plot of column 2 over column 1, log Y axis
uplot line -H -d, --ylim 0.01,1000 --ylog timings.csv

# Box plot per group (column 1 = label, column 2 = value)
uplot box -H -d, latencies.csv

# Density plot — quick sanity check for a distribution
uplot density -H -d, samples.csv

# Heat-map of a matrix on stdin (rows = lines, cols = fields)
uplot colors          # show palette
uplot heatmap -d, --cmap viridis matrix.csv

# Live updating bar chart — redraws on every new stdin line
tail -f /var/log/access.csv | awk -F, '{print $2}' \
  | uniq -c | uplot bar -s

# Pipe straight from a SQL query
psql -A -F, -t -c 'select status, count(*) from req group by 1' \
  | uplot bar -d,
```

## Pipeline patterns this enables

- **Throw-away EDA over SSH**: no need to `scp` a CSV to your
  laptop and open Jupyter — `ssh prod 'psql ... | uplot
  hist'` answers "is the latency distribution bimodal?" in one
  round-trip.
- **CI artifacts that read like plots**: emit `uplot bar`
  output into the build log so the diff between two
  benchmark runs shows up *visually* in the GitHub Actions
  log viewer, not as a 4-decimal table the reviewer skims
  past.
- **Live dashboards in tmux panes**: `uplot ... -s` in one
  pane next to `htop` and `tail -f` gives you a 3-pane "ops
  console" with no Grafana, no Prometheus, no browser.
- **Composes with the rest of the zoo**: pair with
  [`miller`](../miller/) or [`xan`](../xan/) for the reshape
  step (`mlr --csv stats1 -a mean,p99 -f latency -g endpoint
  | uplot bar -H -d,`) and with [`hyperfine`](../hyperfine/)
  `--export-csv` for benchmark histograms.

## When NOT to use it

- You need **interactive** zoom / pan / hover — use a
  notebook (`matplotlib`, `plotly`) or `gnuplot -p` with an
  X11 / Qt terminal.
- You need **publication-grade** vector output (SVG / PDF
  with anti-aliased curves) — `gnuplot`, `asymptote`, or a
  notebook is the right tool; `uplot` deliberately caps
  resolution at the terminal cell grid.
- Your data is **multi-million-row dense** and you actually
  need every point — Unicode block characters max out around
  one cell per data point; downsample first
  ([`miller`](../miller/), [`xan`](../xan/), `awk`).

## Why it earns a slot in the zoo

There are dozens of terminal plotters; most cover one or two
chart types ([`spark`](https://github.com/holman/spark) does
bar only, `asciigraph` does line only). `youplot` is the
*generalist* — every common plot type, real CSV/JSON parsing
(not just whitespace), streaming redraw, color, and grouping
— in a single `gem install` with no native build step. It is
the answer to "I have a CSV, I have an SSH session, I want
to see a chart **right now**."
