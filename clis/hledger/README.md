# hledger

> **Plain-text double-entry accounting in a single Haskell
> binary** — `hledger` reads a human-editable journal file
> (one transaction per block, balanced debits/credits in
> indented postings) and answers `balance` / `register` /
> `incomestatement` / `balancesheet` / `cashflow` queries
> from the shell, or pops a `hledger-ui` TUI / `hledger-web`
> browser UI over the same data. Pinned to **1.52.1**
> (released 2026-04-28,
> [LICENSE](https://github.com/simonmichael/hledger/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/simonmichael/hledger>

## TL;DR

`hledger` is the answer for anyone who looked at YNAB /
Mint / Quicken / Excel-with-pivot-tables and thought "I want
my finances in a `git`-tracked text file that survives every
SaaS shutdown for the next thirty years." The core data
model is full double-entry accounting (Assets = Liabilities
+ Equity, every transaction balances to zero, accounts form
a colon-separated hierarchy `assets:bank:checking` /
`expenses:food:groceries`), and the journal format is the
same one `ledger` invented in 2003 — which means the same
file can be read by `hledger`, `ledger`, `beancount` (with a
small converter), and any text editor on any OS. Daily
workflow is: download bank CSVs, run `hledger import` (which
applies CSV-to-journal rules from `*.csv.rules` files to
classify each line into accounts), commit the journal to
git, then `hledger balance ^expenses:food --monthly --tree`
or `hledger register assets:bank:checking -p 2026-04` to
answer questions. Multi-currency, commodity prices,
historical-cost vs market-value reporting, budgets
(`--budget`), forecasting (periodic transactions), and
investment tracking are all in core.

## Install

```bash
# Homebrew (macOS / Linux)
brew install hledger

# Stack (any platform with the Haskell tool stack)
stack install hledger hledger-ui hledger-web

# Cabal (Haskell)
cabal install hledger hledger-ui hledger-web

# Pre-built binaries (Linux / macOS / Windows)
# Download from https://github.com/simonmichael/hledger/releases/tag/1.52.1

# Docker
docker pull dastapov/hledger:1.52.1
```

## When to choose hledger

- Long-horizon financial records that must outlive any
  vendor — plain text + git wins by default.
- The reporting question is custom (`balance` by tag, by
  payee, by region; `register` filtered by regex; pivot by
  month × account) and a SaaS dashboard cannot express it.
- Multi-currency / commodity holdings (stocks, crypto,
  foreign-currency accounts) — first-class in core, not an
  add-on.
- The team / household wants reviewable diffs on financial
  data (a journal-file commit is a code review of every
  transaction added since the last sync).
- Migration from `ledger` — same file format, faster on
  large journals, friendlier error messages, plus the
  `hledger-ui` TUI and `hledger-web` browser UI as siblings.

## When to pick something else

- The user wants a polished mobile app with bank
  integrations and AI categorisation — pick a SaaS
  (YNAB, Monarch, Lunch Money) and accept the lock-in.
- Pure cash-flow tracking with no double-entry mental model
  — a spreadsheet is lighter.
- Heavy Python integration / programmatic queries against
  the journal — [`beancount`](https://github.com/beancount/beancount)
  is the Python-native sibling with a richer plugin ecosystem
  (Fava web UI, custom importers in Python).
- Stick with [`ledger`](https://www.ledger-cli.org/) when
  the existing tooling, scripts, and reports are already
  ledger-shaped and the speed difference does not matter.

## Caveats

- The CSV import rules language is powerful but takes one
  evening to learn — start from the
  [example rules in the docs](https://hledger.org/hledger.html#csv-format)
  rather than from scratch.
- `hledger` is intentionally read-mostly from the CLI — new
  transactions are added by editing the journal in `$EDITOR`
  (`hledger add` exists but most users prefer text editing).
  Pair with `hledger-ui` / `hledger-web` / `hledger-iadd`
  for an interactive add flow.
- Performance on million-transaction journals is fine but
  not instant; split the journal by year with `include`
  directives if startup time becomes a problem.
- Reports default to *transaction date* — historical
  cost-basis vs market-value reporting needs explicit price
  directives (`P 2026-04-30 AAPL $200.00`) in the journal
  or a separate prices file.
