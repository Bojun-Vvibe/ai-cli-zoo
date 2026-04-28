# numbat

- **Repo:** https://github.com/sharkdp/numbat
- **Version:** v1.23.0 (2026-02-08)
- **License:** MIT or Apache-2.0 ([LICENSE-MIT](https://github.com/sharkdp/numbat/blob/master/LICENSE-MIT), [LICENSE-APACHE](https://github.com/sharkdp/numbat/blob/master/LICENSE-APACHE))
- **Language:** Rust
- **Install:** `brew install numbat` · `cargo install numbat-cli` · prebuilt binaries on the GitHub release page · binary name is `numbat`

## What it does

`numbat` is a **statically-typed scientific calculator with first-class
units of measure** by `@sharkdp` (also the author of `bat`, `fd`,
`hyperfine`). It is shaped like a desktop `bc` / `qalc` REPL, but the
type system tracks **physical dimensions** through every operation:
`30 km/h * 5 min` evaluates to `2.5 km`, not "750" with no idea what
that number means. Unit conversion is part of the language
(`30 mph -> km/h`), as are user-defined units (`unit fortnight =
14 days`), constants from CODATA (`speed_of_light`, `planck_constant`),
trigonometry that rejects non-angle inputs at type-check time
(`sin(5)` errors, `sin(5°)` works), date / time arithmetic
(`now() + 3 weeks -> "RFC3339"`), and currency conversion against
live ECB rates (`100 EUR -> JPY`).

The CLI runs in three modes: a colourful REPL with persistent
history and tab completion, a one-shot `-e` evaluator for shell
pipelines, and a script runner for `.nbt` files. There is also a
WASM build that powers the [numbat.dev](https://numbat.dev) web
playground if you want to share a calculation as a URL instead of
a Slack screenshot of a Spotlight search.

## When to pick it / when not to

Reach for `numbat` when **your "calculator" question has units in
it** and getting the units wrong would be embarrassing or expensive:
infra capacity planning ("how many requests per second is 12 TB/day
of egress?"), cloud cost back-of-envelope ("$0.023/GB-month *
500 GB * 12 months"), physics homework, cooking conversions, or
any back-and-forth where you keep asking "wait, was that GiB or GB,
mb or Mb, ms or μs?". The type checker turns those silent errors
into a loud refusal, which is exactly what you want.

Skip it for plain integer arithmetic (`bc`, your shell's `$(( ))`,
or `python -c` are shorter). Skip on a remote machine where you
just need a single number — `numbat -e '...'` works but a quick
`echo $(( 60 * 60 * 24 ))` is faster to type. Skip if you need
symbolic algebra; numbat is a calculator, not a CAS — reach for
SymPy / Maxima for that.

## Why it matters in an AI-native workflow

LLMs are notoriously bad at units. "How long until the disk fills
at 4 MB/s of writes if we have 800 GB free?" gets answered
confidently and wrong about a third of the time, with the kind of
factor-of-1024 or factor-of-60 error that nobody catches in code
review. Piping the same question through `numbat -e
'800 GB / (4 MB/s) -> days'` returns one answer with units attached
and the type checker as a referee. It also makes a great **agent
tool** — wire `numbat -e <expr>` behind an MCP "calculator" tool
and the model gets a unit-aware sandbox that refuses nonsense
inputs instead of silently agreeing with itself, which is a
non-trivial correctness improvement on any agent that does capacity
planning, pricing, or physics-flavoured reasoning.

## Example invocations

```bash
# Open the REPL
numbat

# One-shot evaluation from the shell
numbat -e '30 mph -> km/h'
numbat -e '800 GB / (4 MB/s) -> hours'
numbat -e 'speed_of_light * 1 year -> light_years'

# Currency conversion against live ECB rates
numbat -e '100 EUR -> USD'

# Date / time arithmetic
numbat -e 'now() + 90 days -> "RFC3339"'
numbat -e '"2026-12-25T00:00:00Z" - now() -> days'

# Run a script (.nbt file)
numbat ./capacity-plan.nbt

# Inside a script you can declare your own units / functions:
#   unit token = 1
#   unit ktok = 1000 token
#   fn cost(t: ktok) -> USD = t * 0.003 USD/ktok
#   print(cost(1500 ktok))
```

## Alternatives in this catalog

- [`pastel`](../pastel/) — sibling sharkdp tool for colour math;
  same author, same "real types for a domain humans get wrong" vibe,
  applied to RGB / HSL / Lab instead of physical units.
- [`hyperfine`](https://github.com/sharkdp/hyperfine) — another
  sharkdp CLI that quantifies things humans normally eyeball;
  pair it with numbat when you want benchmark output reasoned
  about in proper units.
- [`bc`](https://www.gnu.org/software/bc/) — POSIX arbitrary-precision
  calculator; pick `bc` for pure-number arithmetic, numbat the
  moment a unit shows up.
