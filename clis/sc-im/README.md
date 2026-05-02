# sc-im

> **A ncurses spreadsheet for the terminal with vim keybindings**
> — open a CSV/TSV/XLSX in a real grid, navigate with `hjkl`,
> enter formulas (`@sum`, `@avg`, `@stddev`, ...), pivot, plot,
> and save back out, all inside a 5 MB C binary that has been
> stable since 2017. Pinned to **v0.8.5**
> ([LICENSE](https://github.com/andmarti1424/sc-im/blob/master/LICENSE),
> BSD-4-Clause).

Source: <https://github.com/andmarti1424/sc-im>

## TL;DR

`sc-im` (the **S**pread**s**heet **C**alculator **Im**proved) is
the spiritual successor to the venerable `sc` from 1981, rewritten
with vim modes, Unicode, real `.xlsx` import/export via libxlsxwriter,
and a scripting layer in Lua. Once you press `Enter` on the file
list it drops you into a familiar grid: rows numbered down the
left, columns lettered across the top, a formula bar at the top,
and a status line at the bottom showing the current cell's raw
value, format, and computed value. Movement and editing follow vim
conventions exactly — `h/j/k/l` to move, `=` to enter a numeric
formula, `<` for a left-aligned label, `>` for right, `\` for
center, `dd`/`yy`/`p` to delete/yank/paste rows, `:w` to save,
`:q` to quit. Range selection uses visual mode (`v`).

For analysts who live in the terminal it solves a real problem:
`visidata` is great for exploration but is column-stream-oriented,
not cell-oriented; `mlr`/`miller` is for pipelines, not interactive
formula editing. `sc-im` is the only TUI in this zoo where you can
type `=@sum(A1:A100)` into B1 and see it recompute live as you
edit the column above.

## Install

```bash
# Homebrew (macOS / Linux)
brew install sc-im

# Debian / Ubuntu
apt install sc-im

# Arch Linux
pacman -S sc-im

# From source (needs ncursesw + libxlsxwriter)
git clone https://github.com/andmarti1424/sc-im
cd sc-im/src && make && sudo make install
```

## Typical usage

```bash
# Open a CSV (sc-im auto-detects delimiter)
sc-im budget.csv

# Open an .xlsx workbook
sc-im quarterly.xlsx

# Inside sc-im:
#   h j k l        move one cell
#   gg / G         top / bottom of column
#   ^ / $          start / end of row
#   =@sum(A1:A20)  formula in current cell
#   ="Total"       string in current cell (left-aligned: <"Total")
#   :w             save in current format
#   :w! out.tsv    export as TSV
#   :w! out.xlsx   export as XLSX (needs libxlsxwriter at build)
#   :q             quit
#   Ctrl-r r       recalc all formulas
#   :sort A1:Z999 "+A -B"   sort range by col A asc, col B desc

# Read a CSV from stdin (useful in pipelines)
psql -c "copy (select * from sales) to stdout csv" | sc-im /dev/stdin
```

## Why pick `sc-im`

- **Real interactive formulas.** Unlike `visidata`, `mlr`, or
  `csvkit`, you can type `=@sum(B2:B50)/@count(B2:B50)` into a
  cell and watch it update live. This is the workflow spreadsheet
  users expect.
- **Vim modal editing.** Insert, command, visual, and normal modes
  all behave the way vim users expect. No mouse needed.
- **Reads `.xlsx` directly.** No conversion step — `sc-im
  workbook.xlsx` opens the first sheet; `:nextsheet`/`:prevsheet`
  walk between them.
- **Scriptable in Lua.** `:lua` runs Lua snippets against the live
  sheet, enabling custom transforms without leaving the TUI.
- **Tiny and old-school stable.** No Electron, no Python runtime,
  no JIT — just ncurses and libxlsxwriter. Has been the canonical
  terminal spreadsheet for nearly a decade.
