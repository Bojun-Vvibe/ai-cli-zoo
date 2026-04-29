# fend

> **Arbitrary-precision, unit-aware command-line calculator**
> — a single Rust binary that evaluates expressions like `5
> GiB / 12 min to MB/s` or `sin(1 rad) to deg` with exact
> rational arithmetic. Pinned to **v1.5.8** (commit
> `74f325f8df9d3188c1e691dc830c1e9093783530`,
> [LICENSE.md](https://github.com/printfn/fend/blob/main/LICENSE.md),
> MIT).

Source: <https://github.com/printfn/fend>

## TL;DR

`fend` is the calculator the shell never had: enter
`5'10" to cm` and it answers `177.8 cm`; enter `0.1 + 0.2`
and it answers `0.3` exactly (rational, not a float); enter
`1 GB to GiB` and it knows `GB = 10^9` and `GiB = 2^30`.
It ships unit definitions for SI, binary prefixes, currencies
(refreshed at startup with `--exchange-rates` opt-in),
imperial, time, frequency, data, and angles, and it parses
natural human notation (`5 ft 10 in`, `90°`, `1/3`,
`0xff`, `0b1010`, `2^32`). Two modes: one-shot
(`fend '2^16 - 1'`) for piping into shell scripts, or
interactive REPL with readline-style editing, history,
variable bindings (`x = 42`), and lambda functions
(`\x.x*2`). One static binary, no Python, no Node.

## Install

```bash
# Homebrew (macOS / Linux)
brew install fend

# Cargo
cargo install --locked fend

# Pre-built binary from a release
curl -Lo fend "https://github.com/printfn/fend/releases/download/v1.5.8/fend-v1.5.8-aarch64-apple-darwin"
chmod +x fend && sudo mv fend /usr/local/bin/

# Arch
pacman -S fend

# Nix
nix-env -iA nixpkgs.fend

# verify
fend --version    # 1.5.8
```

Run `fend` with no args for the REPL; pass an expression as
an argument for one-shot evaluation.

## License

MIT — see
[LICENSE.md](https://github.com/printfn/fend/blob/main/LICENSE.md).
Permissive; bundle the binary, vendor the source, ship in a
container — no copyleft obligations.

## One Concrete Example

```bash
# 1. unit conversion in one shot
fend '5 GiB / 12 min to MB/s'
# 7.456540444444... MB / s

# 2. exact rational arithmetic — no float drift
fend '0.1 + 0.2'
# 0.3

# 3. mixed imperial → metric
fend "5'10\" to cm"
# 177.8 cm

# 4. trig with explicit units
fend 'sin(30 deg)'
# approx. 0.5

# 5. data + time → throughput
fend '4 TB / (24 hours) to MiB/s'
# approx. 44.2092091... MiB / s

# 6. interactive REPL with bindings
fend
# > x = 1.21 GW
# > x to hp
# 1622498.31... hp
# > exit

# 7. base conversion
fend '0xdeadbeef to binary'
# 0b11011110101011011011111011101111

# 8. inside a shell pipeline
echo "Used $(df -k /Users | tail -1 | awk '{print $3}') KiB" \
  | sed "s|.*Used \([0-9]*\).*|fend '\1 KiB to GiB'|" \
  | sh
```

## Niche It Fills

**The pocket calculator that understands units and exact
arithmetic.** `bc` is precise but unit-blind; `python -c`
is unit-blind and float-by-default; `units(1)` does units
but not arbitrary-precision rationals or trig. `fend` is the
single binary that handles "convert this throughput", "what
is 0.1 + 0.2 actually", and "how many seconds in a fortnight"
without you reaching for a notebook or `pint`.

## Why use it

Three things `fend` does that the shell stack does not:

1. **Unit-aware throughout.** Every number can carry a unit;
   division and multiplication propagate units (`GiB / s`),
   the `to` operator converts (`GiB/s to Mbps`), and the
   parser accepts mixed forms (`5 ft 10 in`, `1h 30min`).
2. **Exact rationals by default.** `1/3 + 1/3 + 1/3` is
   exactly `1`, not `0.999...`. Floats are opt-in via
   explicit decimal or `to float`. The arithmetic engine is
   arbitrary-precision; no silent overflow at 2^63.
3. **One static binary, REPL or one-shot.** `fend 'expr'`
   for shell pipelines, bare `fend` for an interactive
   session with history and variable bindings. No runtime,
   no `pip install`, no JVM.

## Vs Already Cataloged

- **Vs [`numbat`](../numbat/):** sibling — both are
  unit-aware Rust calculators with REPLs. Numbat is a
  full *language* (custom unit definitions, dimensional
  analysis as a type system, named functions, modules); fend
  is a *calculator* (concise expressions, built-in unit
  table, no module system). Reach for numbat when you want
  to define your own unit system or write a small physics
  script; reach for fend when you want `fend '5 GiB / 12
  min'` to just work.
- **Vs `bc` / `dc` (POSIX):** complementary — `bc` is on
  every Unix box and scripts depend on it; `fend` is what
  you install on your laptop for interactive use. Don't
  replace `bc` in production scripts.
- **Vs `python -c` / `python3 -q`:** Python is more
  general (data structures, libraries) but float-by-default
  and unit-blind without `pint`. `fend` is one keystroke
  shorter and unit-aware out of the box.
- **Vs `qalc` (Qalculate CLI, not cataloged):** sibling —
  qalc is the GTK Qalculate's CLI front end with a deeper
  unit / currency / physical-constants database and symbolic
  math. fend has the cleaner single-binary install and the
  more readable error messages; qalc has the richer database
  if you need exotic constants or symbolic integration.

## Caveats

- **Currency exchange rates are opt-in and network-fetched.**
  By default fend ships no live currency conversion; pass
  `--enable-exchange-rates` (or set in config) to have it
  fetch ECB rates at startup. Air-gapped boxes should leave
  this off; the rates persist in a local cache once fetched.
- **Not a CAS.** No symbolic integration, no equation
  solving, no algebraic simplification of unknowns. `fend`
  evaluates expressions to numbers; for symbolic math reach
  for SageMath or SymPy.
- **REPL history is in `~/.local/state/fend/` (Linux) or
  the equivalent on macOS / Windows.** Useful to know if
  you ever paste a secret into the calculator by mistake.
- **Custom units require editing config (`fend --config`).**
  The built-in table is broad but not extensible at the
  CLI; if you need to define many domain units, numbat's
  module system is a better fit.
- **Output precision defaults to truncated decimal for
  irrationals.** Append `to fraction` for exact rational
  output, or set `--digits N` to widen the decimal display.
