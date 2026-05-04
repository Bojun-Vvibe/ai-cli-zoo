# ledger

> **Plain-text double-entry accounting in a single C++ binary.**
> You write transactions in a flat `.ledger` text file; `ledger`
> parses, validates, and reports on it without any database,
> daemon, or GUI. Pinned to **v3.4.1** (commit
> `059e3ad1fd36039e1873f004a7f1e1f408811ffd`,
> [LICENSE.md](https://github.com/ledger/ledger/blob/main/LICENSE.md),
> BSD-3-Clause).

Source: <https://github.com/ledger/ledger>

## TL;DR

`ledger` is the original "plain-text accounting" tool: your entire
book of accounts is one (or many `include`-d) UTF-8 text files in a
small, line-oriented syntax. Every transaction lists two or more
postings whose amounts must sum to zero across accounts (the
double-entry invariant); `ledger` parses the file on every
invocation, enforces that invariant, and answers reporting
questions — `balance`, `register`, `equity`, `budget`, custom
`--format` queries — directly from the parsed in-memory tree. There
is no `init`, no migration, no schema, no server. The binary is
~7 MB; books with tens of thousands of transactions parse and
report in well under a second on commodity hardware.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ledger

# Linux package managers
# Debian / Ubuntu: apt install ledger
# Fedora:          dnf install ledger
# Arch:            pacman -S ledger
# Nix:             nix-env -iA nixpkgs.ledger

# from source (any OS, requires Boost + cmake)
git clone --branch v3.4.1 https://github.com/ledger/ledger.git
cd ledger && ./acprep update && ./acprep make
sudo ./acprep install

# verify
ledger --version    # Ledger 3.4.1, the command-line accounting tool
```

`ledger` reads `$LEDGER_FILE` by default, or whatever you pass with
`-f path/to/book.ledger`. A new user can be productive with one
file and three commands (`balance`, `register`, `print`); the rest
is reporting sugar.

## License

BSD-3-Clause — see
[LICENSE.md](https://github.com/ledger/ledger/blob/main/LICENSE.md).
Permissive, retain copyright + disclaimer in redistributions; no
patent or trademark grants.

## One Concrete Example

Sample `book.ledger`:

```ledger
; book.ledger — one month of a freelancer's books

2026-04-01 * Opening balances
    Assets:Checking                       $2,500.00
    Equity:Opening Balances

2026-04-03 * Acme Corp
    ; invoice #2026-014, net-30
    Assets:Receivable:Acme                $4,000.00
    Income:Consulting

2026-04-05 * Whole Foods
    Expenses:Groceries                       $87.42
    Assets:Checking

2026-04-15 * Acme Corp — payment received
    Assets:Checking                       $4,000.00
    Assets:Receivable:Acme

2026-04-20 * Quarterly tax estimate
    Expenses:Taxes:Federal:Estimated      $1,200.00
    Assets:Checking
```

Run reports against it:

```bash
# 1. balance sheet — every account, indented hierarchy, totals roll up
ledger -f book.ledger balance
#            $5,212.58  Assets
#            $5,212.58    Checking
#           $-2,500.00  Equity:Opening Balances
#            $1,287.42  Expenses
#               $87.42    Groceries
#            $1,200.00    Taxes:Federal:Estimated
#           $-4,000.00  Income:Consulting

# 2. register — chronological journal, one posting per line, running total
ledger -f book.ledger register Expenses

# 3. only checking-account postings, with running balance
ledger -f book.ledger register Assets:Checking

# 4. P&L for April 2026
ledger -f book.ledger -p "this month" balance ^Income ^Expenses

# 5. budget vs actual (requires a `~ Monthly` periodic transaction in book)
ledger -f book.ledger budget Expenses

# 6. round-trip the parsed file (canonical form — useful as a formatter)
ledger -f book.ledger print > book.canonical.ledger

# 7. machine-readable export for downstream tooling
ledger -f book.ledger --format "%(date),%(payee),%(account),%(amount)\n" \
       register > postings.csv

# 8. enforce the double-entry invariant on a CI check
ledger -f book.ledger --explicit --strict balance >/dev/null \
  || { echo "ledger file does not balance"; exit 1; }
```

## Niche It Fills

**Accounting as a flat text file under version control, parsed on
demand.** GUI / SaaS accounting tools (QuickBooks, Xero, YNAB) own
your data inside a proprietary store, lock changes behind a UI, and
make `git diff`, code review, and automation hard. Spreadsheet-based
bookkeeping has the inverse problem: no balance enforcement, no
account hierarchy, no reporting beyond formulas you wrote yourself.
`ledger` is the third option: human-readable text, machine-checked
double-entry, every report a one-line CLI invocation, the whole
history `git log`-able like source code.

## Why use it

1. **Plain text, no database.** The book is a text file (or set of
   `include`-d files); `git`, `grep`, `sed`, your editor's
   multi-cursor, and any LLM that can edit text are all valid
   tools. There is no migration step when the schema changes,
   because there is no schema beyond the syntax. Backups are
   `cp` and conflict-resolution is `git merge`.
2. **Double-entry invariant enforced at parse time.** Every
   transaction's postings must sum to zero (across commodities
   too — `$10.00` cannot balance against `100 EUR` without an
   explicit conversion). A typo that loses a digit fails loudly
   on the next `ledger balance`, not silently in next quarter's
   P&L. `--strict` and `--explicit` upgrade more soft warnings to
   hard errors for CI use.
3. **Reporting is composable on the CLI.** `balance`, `register`,
   `print`, `equity`, `budget`, `stats` are the verbs;
   account-name regexes, `-p "last month"` period expressions, and
   `--format` templates compose into precise queries without
   writing SQL or exporting CSV. `--format` in particular makes
   `ledger` a clean upstream for any plotting / dashboard / LLM
   pipeline.

For an LLM-CLI workflow, `ledger -f book.ledger --format "..."
register` returns deterministic, line-oriented output that an agent
can parse without HTML scraping or API auth, and `ledger print`
canonicalises the file so an agent's edits round-trip cleanly
through `git diff`.

## Vs Already Cataloged

- **Vs [`hledger`](../hledger/):** same plain-text-accounting
  family, same file format (mostly compatible). `hledger` is the
  Haskell rewrite with a friendlier CLI (`hledger add` interactive
  entry, built-in web UI `hledger-web`, Python-style error
  messages); `ledger` is the original, fastest on very large
  books, and supports a few constructs `hledger` does not (full
  value expressions, automated transactions with arbitrary
  Python-like predicates). Pick `hledger` for ergonomics, `ledger`
  for raw speed and the long tail of advanced syntax. Books are
  largely portable both ways.
- **Vs [`beancount`](../beancount/):** also same family,
  Python-based, stricter syntax (typed directives, explicit
  `open`/`close` for every account, named meta-data on
  postings). `ledger` is permissive — accounts spring into
  existence on first use, commodities are inferred — which is
  great for a personal book but lets typos through that
  `beancount`'s typed parser would catch. Use `beancount` when
  you want a strongly-typed book and a plugin ecosystem
  ([`fava`](../fava/) web UI, importers); use `ledger` when you
  want the smallest, fastest, dependency-free reporting binary.
- **Vs spreadsheets (Excel / Google Sheets / Numbers):**
  spreadsheets have no concept of double-entry, no account
  hierarchy, no period expressions, and merge badly under `git`.
  `ledger` enforces the accounting invariant the spreadsheet
  cannot, at the cost of a ~1-page syntax to learn.

## Caveats

- **Permissive parser, no schema.** A misspelled account name
  (`Asset:Checking` vs `Assets:Checking`) creates a new account
  silently; the balance still sums to zero, but reports split
  across the typo. The cure is `ledger accounts` to spot
  unexpected leaves, plus `--strict` + an explicit `account
  Assets:Checking` declaration block at the top of the file to
  reject undeclared accounts.
- **No built-in importers.** `ledger` reads `.ledger` files only;
  converting bank CSV / OFX / QIF into postings is on you.
  Common patterns are a `Makefile` of `awk`/`python` converters,
  or the third-party [`ledger-autosync`] / [`reckon`] /
  [`icsv2ledger`] tools — none are bundled.
- **Reports are CLI-only; no native web UI.** If you want a
  browseable balance sheet, you either pipe `--format` into your
  own static-site generator or switch to [`beancount`](../beancount/)
  + [`fava`](../fava/) (which is in fact why those projects
  exist). `ledger-web` exists but is third-party and lightly
  maintained.
- **Build from source is heavyweight.** `ledger` depends on
  Boost, MPFR, GMP, and (optionally) Python — the source build
  pulls in ~500 MB of dev packages. Prefer the OS package
  manager unless you need an unreleased feature; the package
  versions track upstream within one or two minor releases on
  most distros.
- **Multi-currency requires explicit prices.** Postings in
  different commodities cannot net to zero unless you provide a
  `P` price directive or an inline `@@` conversion. `ledger` will
  not invent an exchange rate; reports across mixed commodities
  show each commodity separately until a price is supplied. This
  is correct accounting behaviour but surprises new users.
