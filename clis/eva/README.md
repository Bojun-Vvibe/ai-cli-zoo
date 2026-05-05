# eva

> **Calculator REPL for the terminal — `bc` with modern syntax,
> readline, and a real expression language.** Reads infix arithmetic,
> hex/oct/bin literals, scientific notation, named variables, anonymous
> and named functions, vectors / sums, and 30+ built-ins (trig,
> logs, gcd, factorial) at an interactive prompt or as a one-shot
> `eva '<expr>'` arg. Pinned to **v0.3.1**
> ([LICENSE](https://github.com/NerdyPepper/eva/blob/master/LICENSE),
> MIT).

Source: <https://github.com/NerdyPepper/eva>

## TL;DR

`bc` is the POSIX answer for "I need a calculator in the terminal,"
but its prompt is hostile (no readline by default, no syntax
highlighting, no error markers, no recently-used history across
sessions). `eva` is the modern Rust take: live syntax-coloured input,
underline-marker error reporting at the offending column,
multi-session history, named bindings (`x = 42`, `f(n) = n*(n+1)/2`),
and a one-shot mode (`eva '2^32 - 1'`) that fits naturally in shell
pipelines and scripts.

## Install

```bash
# Cargo
cargo install eva

# Arch (community)
pacman -S eva

# Nix
nix-env -iA nixpkgs.eva

# Pre-built binaries:
# https://github.com/NerdyPepper/eva/releases/tag/v0.3.1

# Verify
eva --version    # eva 0.3.1
```

Run `eva` for the REPL or `eva '<expr>'` for one-shot mode.

## License

MIT — see
[LICENSE](https://github.com/NerdyPepper/eva/blob/master/LICENSE).

## One Concrete Example

```bash
# 1. one-shot from the shell — pipeline-friendly
$ eva 'sqrt(2) * pi'
4.442882938158366

# 2. unit conversion via constants
$ eva '0xFFFFFFFF / 1024^3'
3.999999999068677  # bytes → GiB

# 3. quick log-scale check (decibels)
$ eva '20 * log(48000 / 8000) / log(10)'
15.563025007672872

# 4. interactive REPL with named bindings + functions
$ eva
> a = 42
> f(n) = n * (n + 1) / 2
> f(a)
903
> sum(1, 100, k, k^2)   # Σ k=1..100 of k²
338350

# 5. radix conversion via format specifier
$ eva --radix=h '255 + 1'
0x100
```

## Niche It Fills

**Friction-free calculator at the terminal prompt.** The catalog has
[`qalc`](../qalc/) (Qalculate, full-fat unit calculator with currency
+ physical units), [`numbat`](../numbat/) (statically typed scientific
calculator with first-class units),
[`kalker`](../kalker/) (math-textbook syntax with derivatives /
integrals), [`fend`](../fend/) (arbitrary-precision unit-aware), and
[`rink`](../rink/) (units / conversions). `eva` sits at the *opposite*
end of that spectrum: small, fast, no unit system, no symbolic engine
— a `bc` replacement for the 95% of "I just need to do some
arithmetic" moments where the unit-aware tools are overkill.

## Why use it

1. **One-shot ergonomics.** `eva '<expr>'` returns a single line on
   stdout suitable for `$(eva …)` substitution and for embedding in
   `make` / `just` / shell scripts. The unit-aware peers print extra
   formatting that needs stripping.
2. **Readline + history + colour by default.** Up-arrow recalls
   previous expressions across sessions; the input is syntax-coloured
   live; errors point at the offending column with a `^` underline.
3. **Tiny binary, instant startup.** No Qt / GTK / Python / SymPy
   dependency tree to drag in; cold start is single-digit
   milliseconds, which matters when the tool lives at the prompt.

## Vs Already Cataloged

- **Vs [`qalc`](../qalc/):** different weight class. `qalc` is the
  full Qalculate engine with units, currencies, physical constants,
  exact-rational arithmetic. `eva` is a four-function-plus calculator
  with bindings and functions but no units. Pick `qalc` when the
  question is "12.5 USD/h × 37 weeks in EUR"; pick `eva` when the
  question is "what is `sqrt(2) * pi`".
- **Vs [`numbat`](../numbat/) / [`fend`](../fend/) /
  [`rink`](../rink/):** all are unit-aware. `eva` deliberately is
  not, which keeps the surface tiny.
- **Vs [`kalker`](../kalker/):** closest peer in spirit (terminal
  math REPL). `kalker` adds symbolic derivatives, integrals, and
  textbook-style notation; `eva` stays purely numerical with a more
  shell-script-friendly one-shot mode.
- **Vs `bc` / `dc` (POSIX):** the same niche modernised — readline,
  colour, multi-session history, expression-language conveniences
  (named functions, summation), and a less hostile error model.
- **Vs Python / `python -c '…'`:** no interpreter startup cost
  (~5 ms vs ~30 ms), no `import math`, expressions read like math
  not Python (`sqrt(2)*pi` not `math.sqrt(2)*math.pi`).

## Caveats

- **No unit system.** Adding two quantities with implicit units
  ("3 km + 200 m") will silently give 203, not 3200. For unit
  arithmetic use [`numbat`](../numbat/) / [`qalc`](../qalc/) /
  [`fend`](../fend/).
- **Floating-point only.** No exact rationals, no arbitrary-precision
  big-integer mode. `eva '2^200'` returns a float in scientific
  notation, not the exact integer. Use [`fend`](../fend/) /
  [`qalc`](../qalc/) for exact arithmetic.
- **No symbolic engine.** Cannot differentiate, integrate, or solve
  symbolically. Use [`kalker`](../kalker/) for those.
- **Maintenance cadence.** Releases are infrequent; the v0.3.1 tag
  is stable and ships in distros, but expect long gaps between
  feature releases.
