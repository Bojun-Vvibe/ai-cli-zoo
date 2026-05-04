# beancount

> **Strongly-typed plain-text accounting in Python.** A stricter,
> Python-based cousin of `ledger`: every account must be `open`-ed
> before use, every directive is typed, and the parser rejects
> anything ambiguous at load time. Pinned to **v3.2.2** (PyPI;
> repo HEAD `5704a86108948b4dff84cfff816d9a34a0a5da30`,
> [COPYING](https://github.com/beancount/beancount/blob/master/COPYING),
> GPL-2.0).

Source: <https://github.com/beancount/beancount>
PyPI: <https://pypi.org/project/beancount/>

## TL;DR

`beancount` is plain-text accounting for people who want the
parser to catch their mistakes. Unlike `ledger`'s permissive,
"accounts spring into existence" model, `beancount` requires every
account to be explicitly `open`-ed (with a date and currency
constraint), every commodity to be declared, and every transaction
to either balance to zero or use exactly one `Assets:Foo`
auto-balancing posting. The parser is a Bison/Flex grammar wrapped
in Python; once parsed, the entire book is a typed in-memory list
of `Transaction` / `Open` / `Balance` / `Price` / `Pad` / `Note`
records that downstream tools (the bundled `bean-*` CLIs and the
[`fava`](../fava/) web UI) consume. The format is designed for
plugin authorship — anyone can write a Python plugin that
transforms the parsed stream before reporting.

## Install

```bash
# pipx (recommended — isolated, all bean-* commands on PATH)
pipx install beancount

# pip
pip install --user beancount

# uv
uv tool install beancount

# Nix
nix-env -iA nixpkgs.beancount

# verify — beancount ships several bin entry points
bean-check --version       # 3.2.2
bean-query --help
bean-format --help
bean-doctor --help
bean-example --help
```

`beancount` is a *library* with thin CLI wrappers; the typical
day-to-day commands are `bean-check` (parse + validate),
`bean-query` (SQL-like queries), `bean-format` (canonicalise
whitespace), and `bean-doctor` (diagnose specific files / line
ranges). For the browseable web UI, install
[`fava`](../fava/) on top.

## License

GPL-2.0 — see
[COPYING](https://github.com/beancount/beancount/blob/master/COPYING).
Copyleft: derivative works that are distributed must also be
GPL-2.0. Importantly, GPL-2.0 does *not* cover the data file
itself — your `.beancount` ledger remains your own.

## One Concrete Example

Sample `book.beancount`:

```beancount
; book.beancount — typed plain-text book
option "title" "Example Freelance Book"
option "operating_currency" "USD"

2026-01-01 open Assets:Checking          USD
2026-01-01 open Assets:Receivable:Acme   USD
2026-01-01 open Income:Consulting        USD
2026-01-01 open Expenses:Groceries       USD
2026-01-01 open Expenses:Taxes:Federal   USD
2026-01-01 open Equity:Opening-Balances  USD

2026-04-01 * "Opening balance"
  Assets:Checking                       2500.00 USD
  Equity:Opening-Balances

2026-04-03 * "Acme Corp" "Invoice #2026-014"
  invoice: "2026-014"
  due: 2026-05-03
  Assets:Receivable:Acme                4000.00 USD
  Income:Consulting                    -4000.00 USD

2026-04-05 * "Whole Foods"
  Expenses:Groceries                      87.42 USD
  Assets:Checking                        -87.42 USD

2026-04-15 balance Assets:Checking      6412.58 USD
```

Then exercise the toolchain:

```bash
# 1. parse + run all built-in validators (balance assertions, types)
bean-check book.beancount
# (silent on success; prints typed errors with file:line on failure)

# 2. SQL-like query against the parsed book
bean-query book.beancount "SELECT account, sum(position) WHERE account ~ 'Expenses' GROUP BY account"

# 3. canonical formatter — aligns amounts, normalises whitespace
bean-format book.beancount > book.formatted.beancount

# 4. diagnostic — what's happening at this line?
bean-doctor context book.beancount 23

# 5. seed a new book with a believable example you can edit
bean-example > new-book.beancount

# 6. CI gate: fail the build if the book stops type-checking
bean-check book.beancount || { echo "ledger broken"; exit 1; }

# 7. drive the web UI (separate package — see clis/fava/)
fava book.beancount
# → http://localhost:5000  with browseable balance sheet, P&L, journal
```

## Niche It Fills

**The "type-checked compiler" of plain-text accounting.** `ledger`
is the permissive interpreter — anything that parses and balances
is valid. `beancount` is the strict, declarative parser — the file
declares accounts, commodities, and balance assertions up-front,
and any transaction that contradicts the declarations is rejected
before reports run. For a personal book that is going to live
under version control for a decade, the up-front cost (a dozen
extra `open` directives) buys a regression suite for your finances.

## Why use it

1. **Typed parser, mistakes fail at parse time.** A misspelled
   account name (`Assets:Checking` → `Asset:Checking`) is rejected
   immediately because `Asset:Checking` was never `open`-ed. A
   `balance` directive on the wrong date fails loudly with the
   computed-vs-asserted delta. Currency mismatches between
   posting and account fail too. The same class of bugs that go
   silent in `ledger` until reporting time get caught at the next
   `bean-check`.
2. **Plugin architecture in Python.** `option "plugin"
   "beancount.plugins.auto"` and you have ~20 bundled
   transformations (auto-account-opening, implicit prices,
   booking methods, FIFO lot tracking, …); the same hook lets
   you write a 30-line Python plugin that, e.g., enforces "every
   `Expenses:` posting must have a `category:` meta tag" against
   the typed AST. No other plain-text-accounting tool has this
   level of programmability.
3. **Companion ecosystem speaks the same AST.**
   [`fava`](../fava/) (web UI), `bean-query` (SQL),
   `beangulp` / `smart_importer` (CSV → typed transactions),
   `beanprice` (commodity-price downloaders) all consume the same
   parsed `Transaction` records, so a plugin you write upstream is
   visible to every downstream tool. The 3.x line cleaned up the
   public API specifically to make this contract stable.

For an LLM-CLI workflow, `bean-check` is the validation gate (book
either type-checks or it doesn't, with a precise file:line on
failure), and `bean-query` produces deterministic tabular output
that an agent can pipe straight into a chart or a follow-up
prompt without HTML scraping.

## Vs Already Cataloged

- **Vs [`ledger`](../ledger/):** sibling tools in the same
  family. `ledger` is permissive (accounts auto-create,
  commodities inferred); `beancount` is strict (everything
  declared, parser rejects ambiguity). `ledger` is a single
  ~7 MB C++ binary, `beancount` is a Python package with several
  `bean-*` entry points. Pick `beancount` when you want the
  parser to be your safety net and you're comfortable with
  Python tooling; pick `ledger` when you want one binary and
  zero ceremony. Files are largely portable (with mechanical
  syntax conversion).
- **Vs [`hledger`](../hledger/):** `hledger` is the Haskell
  member of the same family — friendlier than `ledger`, less
  strict than `beancount`. `hledger`'s `add` interactive entry
  and `hledger-web` UI are the closest peer to `bean-query` +
  `fava`, but the type system stays at the "balance to zero"
  level rather than "every account declared". Use `hledger` for
  ergonomics, `beancount` for strictness + plugins.
- **Vs [`fava`](../fava/):** not a competitor —
  [`fava`](../fava/) is the web UI built on top of
  `beancount`'s parsed AST. `fava` does not parse `.beancount`
  files itself; it imports `beancount.loader` and renders the
  same record stream `bean-query` operates on. Most users
  install both.

## Caveats

- **Strictness is a learning curve.** A new user copying examples
  from blog posts will trip over "account not open on this date",
  "commodity USD not declared", and "transaction does not
  balance and has no auto-balancing posting" within the first
  hour. The wins arrive on month two, when you stop making the
  errors at all.
- **GPL-2.0, not MIT/BSD.** If you ship a derivative work
  (e.g. a closed-source SaaS that statically links the parser),
  the GPL applies. For the common case of editing your own book,
  this is irrelevant — your `.beancount` data file is your own
  IP. Companion tools published under MIT (`fava` is MIT) help
  keep the redistributable surface clean.
- **Python performance ceiling.** Books over ~100k transactions
  noticeably slow `bean-check` and `bean-query` (seconds, not
  sub-second like `ledger`). For most personal and freelance
  books this is invisible; for enterprise-scale general-ledger
  data, prefer `ledger` (or a real accounting system).
- **No release tags on GitHub.** Versions are published to PyPI
  only — `pip install beancount==3.2.2` is the canonical pin, and
  the repo does not publish matching `v3.2.2` git tags. Pin the
  PyPI version in CI rather than a git ref.
- **Importers are not bundled.** `beangulp` (the CSV → typed
  transactions framework) is a separate project; you write a
  small Python class per bank account that converts statements
  into `Transaction` records. The pattern is well-documented but
  it is not "plug in your bank URL and go".
