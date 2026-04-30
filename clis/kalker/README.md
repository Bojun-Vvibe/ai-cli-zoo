# kalker

> **A scientific calculator REPL with arbitrary-
> precision arithmetic, units, vectors, complex
> numbers, derivatives, integrals, sums, and
> user-defined functions** — a single Rust binary
> (`kalk`) that lets you write
> `integrate(0, pi, sin(x) dx)` and get an exact
> answer at the prompt. Pinned to **v2.2.2**
> ([LICENSE](https://github.com/PaddiM8/kalker/blob/master/LICENSE),
> MIT).

Source: <https://github.com/PaddiM8/kalker>

## TL;DR

`kalker` (binary name `kalk`) is a calculator
with the convenience of `bc` and the math
surface of a scientific TI calculator. The REPL
parses **infix math the way you'd write it on
paper** — implicit multiplication (`2x`, `3sin
x`), `pi`, `e`, `i` as built-ins, suffix units
(`km`, `kg`, `B`, `KiB`), and base prefixes
(`0x`, `0b`, `0o`). It supports **arbitrary-
precision numbers** (compile-time flag enables
`rug`-backed bignums; the bundled binary uses
floating-point but with correct rounding for
common functions), **complex arithmetic** out
of the box (`(1+2i) * (3-i)` works), **vectors
and matrices** (`[1,2,3] dot [4,5,6]`),
**equation solving** (`5x + 3 = 18` returns `x =
3`), and **calculus operators** for derivatives
(`f'(2)`), integrals (`integrate(a, b, expr
dx)`), and sums (`sum(n=1, 10, n^2)`). You can
**define your own functions** with `f(x) = x^2
+ 1` and they persist for the session (write
them to `~/.config/kalker/kalker.kalk` to load
on every start). Output rendering supports
**LaTeX export** (`--input-mode tex`) and a
small built-in WebAssembly build runs the same
engine in browsers, so a snippet that works in
your terminal also works in a static math doc.

## Install

```bash
# Homebrew
brew install kalker

# Cargo (pulls and builds from crates.io)
cargo install kalker

# Arch Linux
sudo pacman -S kalker

# prebuilt binaries (Linux / macOS / Windows / Android)
# https://github.com/PaddiM8/kalker/releases

# verify
kalk --version    # kalker 2.2.2
```

## Basic usage

```text
$ kalk
>> 2 + 3 * 4
14

>> sqrt(2)
1.4142135624

>> 5km + 300m to mi
3.2932739419 mi

>> (1 + 2i) * (3 - i)
5 + 5i

>> integrate(0, pi, sin(x) dx)
2

>> f(x) = x^2 + 2x + 1
>> f(3)
16

>> sum(n=1, 100, 1/n^2)
1.6349839002
```

```bash
# one-shot mode (no REPL) for scripts
kalk "sin(pi/4) * sqrt(2)"     # -> 1

# read expression from stdin
echo "log(1024, 2)" | kalk
```

## When to choose

- **You want a desktop calculator inside the
  terminal that handles units and complex
  numbers** — `bc` doesn't, `python -c` requires
  imports for any of it, kalker just works at the
  prompt.
- **You need symbolic-feeling answers
  occasionally** — derivatives, integrals, and
  sums numerically evaluated at the prompt cover
  the back-of-envelope use cases without booting
  Mathematica.
- **You define helper functions for engineering
  back-of-envelope math** — `f(x) = ...` plus a
  loadable `kalker.kalk` file gives you a small
  personal math library that travels with your
  dotfiles.

## When NOT to choose

- **You need symbolic CAS** (factor polynomials,
  expand series, manipulate algebraic
  expressions) — use SymPy / Mathematica /
  Maxima; kalker is a numerical evaluator with a
  few calculus operators, not a symbolic engine.
- **You want a unit-first calculator with a
  units database the size of GNU `units`** — use
  [`numbat`](../numbat/), which is unit-checked
  by design; kalker handles common units but
  isn't a dimensional-analysis system.
- **You want plotting** — kalker prints
  numbers; pipe values into [`gnuplot`](../gnuplot/)
  or use a notebook for graphs.

## Why it fits the zoo

Joins the "small Rust calculator at the
prompt" cluster alongside [`numbat`](../numbat/)
(units-first, dimensionally checked) and
[`fend`](../fend/) (units-first conversion
calculator). kalker's specific gap is **scientific
+ calculus + vectors + complex + user-defined
functions in one REPL** — numbat is stricter on
units, fend is conversion-leaning, kalker is the
"general-purpose programmable calculator" of the
three.

## Upstream pointers

- Repo: <https://github.com/PaddiM8/kalker>
- Release notes: <https://github.com/PaddiM8/kalker/releases>
- License: [MIT](https://github.com/PaddiM8/kalker/blob/master/LICENSE) (`LICENSE`)
- Maintainer: [@PaddiM8](https://github.com/PaddiM8)
