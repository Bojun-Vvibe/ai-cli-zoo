# bagels

> **Terminal expense / personal-finance tracker** — a Python +
> Textual TUI for double-entry-style personal accounting that
> stores everything in a local SQLite file: accounts (cash,
> checking, credit cards, brokerages), categories (rent, food,
> salary, dividends), records (income / expense / transfer
> between accounts) with paid-by + splits-among person
> tracking, recurring-transaction templates, budget envelopes
> with running balances, monthly / category insights with
> ASCII charts, and an actual-budget importer — pinned to
> **0.3.12** (commit
> [`557a648`](https://github.com/EnhancedJax/Bagels/commit/557a648f69fde06f779734a5057df91e822416aa),
> [LICENSE](https://github.com/EnhancedJax/Bagels/blob/0.3.12/LICENSE),
> GPL-3.0-only).

Source: <https://github.com/EnhancedJax/Bagels>

## TL;DR

Personal-finance tools sit in three buckets: **GUI desktop
apps** (GnuCash, Banktivity, Money.app — full double-entry
but heavyweight, mouse-driven, no SSH story), **SaaS web
apps** (YNAB, Lunch Money, Actual — slick but vendor-locked
and metered), and **plain-text accounting CLIs**
([`hledger`](../hledger/), [`beancount`](../beancount/),
`ledger` — diff-friendly text journals with industrial-grade
double-entry rigour but a high learning curve and no
interactive entry surface).

`bagels` is the missing fourth pick: a **TUI** with the
keyboard-only ergonomics of the plain-text CLIs but the
hand-holding of a SaaS app — categorised dropdowns, an
autocomplete record-entry modal, a real account-balance
view, recurring transactions as templates instead of
repeated manual entries, and a calendar widget that shows
spend-per-day at a glance. The data lives in one SQLite
file under `$XDG_DATA_HOME/bagels/` so backup is `cp` and
analytics is `sqlite-utils memory bagels.db "select ..."`.

The killer property is **paid-by + splits-among on every
record**: a "$120 dinner, paid by me, split 4 ways" entry
generates the right balance changes against a "Friends"
person ledger automatically — a workflow GnuCash forces
into manual offsetting transactions and that hledger
handles only with hand-rolled multi-posting entries. For
operators who travel with friends, share rent, or run a
small expense-shared household, this single feature is
worth the install.

## Install

```bash
# pipx (recommended — isolated venv)
pipx install bagels

# pip
pip install bagels

# Then one-time init (interactive: picks default accounts +
# categories from a starter template, or starts blank)
bagels
```

Requires Python 3.10+. The Textual UI works on any modern
terminal (`xterm`, `alacritty`, `kitty`, `wezterm`, the
default macOS / GNOME / KDE terminals); colour + mouse
support is opportunistic (graceful 256-colour fallback).
Cross-platform: Linux, macOS, Windows (PowerShell + Windows
Terminal).

## Example usage

```bash
# launch the TUI
bagels

# specify a non-default DB location
bagels --db ~/Documents/finances/bagels.db

# import an Actual Budget export (added in 0.3.6)
bagels --import-actual ~/Downloads/actual-export.zip

# print version
bagels --version
```

In-TUI keymap (subset):

- `n` new record (modal: account, category, amount, paid-by,
  splits, label, date)
- `t` create from template
- `r` recurring-transaction editor
- `c` calendar view (per-day spend, click for detail)
- `i` insights tab (monthly / category breakdowns + ASCII
  charts via `textual-plotext`)
- `b` budgets tab (envelope balances)
- `m` manager (categories + people CRUD)
- `Tab` cycle main panels
- `?` help, `q` quit

Configuration lives at `$XDG_CONFIG_HOME/bagels/config.yaml`
(theme, currency symbol, date format, hotkey overrides).

## Why it matters

Most operators *want* to track expenses but bounce off the
two extremes: SaaS apps cost $80–$120/year and lock the
data in a cloud the operator does not control; plain-text
accounting requires writing journals like

```
2026/05/04 * Dinner with Alice + Bob
    Expenses:Food:Restaurants    $120.00
    Liabilities:Receivable:Alice  $-40.00
    Liabilities:Receivable:Bob    $-40.00
    Assets:Checking              $-40.00
```

by hand for every transaction — the right answer for an
accountant, the wrong on-ramp for someone who wants to
know how much they spent on coffee last month.

`bagels` is the in-between: keyboard-fast, SQLite on disk
(diffable via `sqlite-utils diff` if not line-by-line
mergeable), GPL-3.0 OSS, and the entry modal does the
double-entry posting under the hood so the user never
sees a debit / credit pair. Especially valuable for:

- A tiling-WM Linux desktop where opening an Electron
  budget app is a 200 MB cost.
- A laptop on a long flight (no internet, no SaaS).
- An operator who already lives in `tmux` + `vim` and
  resents context-switching to a GUI for one expense.
- A privacy-sensitive household where bank statements are
  reconciled manually instead of via Plaid / SaltEdge OAuth.

Pairs with [`hledger`](../hledger/) (industrial double-entry
on top of plain text — many operators do daily entry in
`bagels` and run a quarterly export through `hledger` for
tax-grade reports), with [`sqlite-utils`](../sqlite-utils/)
(`sqlite-utils memory ~/.local/share/bagels/bagels.db
"select category, sum(amount) from records where ..."` is
the canonical ad-hoc query path that no GUI app exposes),
with [`harlequin`](../harlequin/) (interactive SQL IDE over
the same SQLite file when the question is more complex
than one `sqlite-utils` line), with [`datasette`](../datasette/)
(read-only HTTP front-end over the bagels DB for sharing
with a partner / accountant without giving them the bagels
binary), and orthogonal to bank-aggregator pipelines
([`actualbudget/actual`](https://github.com/actualbudget/actual)
self-hosted SaaS — bagels can import an Actual export via
`--import-actual`, so the migration path in either
direction is a one-shot file copy).

## License

GPL-3.0-only on the bagels code itself. See
[LICENSE](https://github.com/EnhancedJax/Bagels/blob/0.3.12/LICENSE)
in upstream. The GPL-3.0 obligations apply only to
redistribution of `bagels` itself — running it against your
own SQLite ledger imposes no obligations on the data.

## As of

2026-05-04. Upstream tag `0.3.12` (latest GitHub release on
`EnhancedJax/Bagels`, released 2025-07-06). The DB schema
has migrated several times across the 0.2 → 0.3 line —
upstream ships migrations on first launch, but back up the
SQLite file before upgrading across minor versions in any
production-shaped use.
