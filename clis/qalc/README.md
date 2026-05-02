# qalc

> **Powerful CLI calculator with units, currencies, symbolic
> math, and live currency rates** —
> the command-line front-end of the Qalculate! engine: type
> `5 ft 3 in to cm`, `(25 USD + 10 EUR) to JPY`, `solve(x^2 = 3x
> + 4, x)`, or `integrate(sin(x)^2, x)` and get the right answer,
> with units, with arbitrary precision, with interval arithmetic
> if you need it.
> Pinned to **v5.10.0**
> ([COPYING](https://github.com/Qalculate/libqalculate/blob/master/COPYING),
> GPL-2.0-or-later).

Source: <https://github.com/Qalculate/libqalculate>

## TL;DR

`qalc` is the readline CLI shipped with `libqalculate`. Underneath
sits a 20-year-old C++ math engine with first-class units (SI +
imperial + obscure ones — `furlong`, `slug`, `parsec`),
~170 currencies with daily-refreshed exchange rates fetched from
ECB / Mycurrency.net (`-e` to update), symbolic math
(differentiation, integration, equation solving, simplification),
arbitrary-precision arithmetic, complex numbers, vectors and
matrices, interval arithmetic for uncertainty propagation, and
a plain-text "calculate-as-you-type" expression parser that DWIMs
natural input (`5'3" to cm`, `25%`, `30C to F`, `now + 90 days`,
`speed of light * 1 ns to m`). The same engine powers the
Qalculate GTK / Qt GUIs; `qalc` is the same brain in 200 KB of
binary that runs over SSH.

## Install

```bash
# Homebrew (macOS / Linuxbrew) — installs libqalculate + qalc
brew install libqalculate

# Debian / Ubuntu
sudo apt install qalc

# Arch
sudo pacman -S libqalculate

# Fedora
sudo dnf install libqalculate

# from source (autotools)
git clone https://github.com/Qalculate/libqalculate && cd libqalculate
./autogen.sh && ./configure && make -j && sudo make install

# verify
qalc -v    # 5.10.0
```

## Examples

```bash
# One-shot mode — answer goes to stdout, exits
qalc '(25 USD + 10 EUR) to JPY'
qalc '5 ft 3 in to cm'
qalc 'solve(x^2 = 3x + 4, x)'
qalc 'integrate(sin(x)^2, x)'

# Refresh exchange rates (run weekly via cron)
qalc -e

# Interactive REPL with calculate-as-you-type
qalc
> c                          # speed of light
> _ * 1 ns to m              # `_` = previous result
> base 16                    # switch output base
> 255 + 1
```

## Niche / category

Calculator — the heavyweight CLI calculator. Not just a `bc`
replacement: units, currencies, symbolic math, plotting (via
gnuplot), and an actual expression parser that handles natural
notation are all in scope.

## When to use

- You want a calculator that *understands units* — converting
  `5 ft 3 in to cm` or `gallon/100mi to L/100km` correctly without
  you looking up the factor.
- Currency math you trust — `(25 USD + 10 EUR) to JPY` with
  rates auto-refreshed from a primary source, not hard-coded.
- Quick symbolic work that doesn't deserve a full Mathematica /
  SymPy session — `solve()`, `diff()`, `integrate()`, `factor()`,
  `simplify()` all from one binary.
- You want it scriptable — `result=$(qalc -t '5 mi to km')`
  returns just the value (`-t` = terse), perfect for shell
  pipelines.

## When NOT to use

- You need `awk`-fast in-stream arithmetic on a million rows —
  `qalc` start-up + parser overhead is fine for one shot,
  not for a hot loop; use `awk` / `bc` / a real language.
- You want a CAS for serious work — `qalc` does symbolic math
  but the depth is far below SymPy / Maxima / Mathematica;
  for research-grade CAS, reach for those.
- You're plotting in a terminal — `qalc` shells out to `gnuplot`
  (which then needs a window or PNG output); for pure-TUI
  plots use [`gnuplot`](../gnuplot/) directly with `set term dumb`,
  or [`youplot`](../youplot/).
- You want offline-only — exchange rate refresh hits the network;
  set `auto-update-exchange-rates 0` in `~/.config/qalculate/qalc.cfg`
  if you need fully offline runs.

## Related

- [`kalker`](../kalker/) — Rust scientific calculator with units
  but no currency / symbolic depth (pick `kalker` for a smaller,
  faster startup and plain numerics; pick `qalc` for units +
  currency + CAS-lite)
- `bc`, `dc` — POSIX calculators (pick when scripting and
  portability matter more than units / parsing)
- [`gnuplot`](../gnuplot/) — what `qalc` shells out to for
  plotting; combine for `qalc 'plot(sin(x)/x, -10, 10)'`
- [`youplot`](../youplot/) — pure-TUI plots from stdin
