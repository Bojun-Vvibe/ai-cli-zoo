# fava

> **Browseable web UI on top of a `beancount` book.** Point `fava
> path/to/book.beancount` at your typed plain-text ledger and get
> a live local web app at `:5000` with balance sheet, P&L,
> journal, holdings, query editor, and import workflow — no
> database, the file is still the source of truth. Pinned to
> **v1.30.12** (commit
> `af81e32a1fcfced0fe04c77373c41df5d31d78e4`,
> [LICENSE](https://github.com/beancount/fava/blob/main/LICENSE),
> MIT).

Source: <https://github.com/beancount/fava>
PyPI: <https://pypi.org/project/fava/>

## TL;DR

`fava` is the missing GUI for [`beancount`](../beancount/). It
imports `beancount.loader`, parses your `.beancount` file on every
request (or watches the file and reloads on change), and renders
the typed AST as a Flask + Svelte web app. You get every report
`bean-query` exposes plus a clickable account tree, a holdings /
commodities view, a built-in query editor with autocomplete, an
import workflow that turns CSV/OFX into draft transactions you
review before they hit the file, and an editor pane that writes
back to the same source `.beancount` files. The app is
single-user, runs on `127.0.0.1:5000` by default, and stores zero
state of its own — closing `fava` and deleting `~/.fava` loses
nothing because the book is the file.

## Install

```bash
# pipx (recommended — isolates fava + beancount in one env)
pipx install fava

# pip
pip install --user fava

# uv
uv tool install fava

# Nix
nix-env -iA nixpkgs.fava

# Docker (official image)
docker run -p 5000:5000 -v "$PWD":/bean yegle/fava /bean/book.beancount

# verify
fava --version       # 1.30.12 (will pull a matching beancount as a dep)

# run it
fava path/to/book.beancount
# → open http://127.0.0.1:5000

# multi-book (separate tabs in the UI)
fava personal.beancount business.beancount

# bind to LAN (read-only by default; see Caveats before exposing)
fava --host 0.0.0.0 --port 5000 book.beancount
```

`fava` requires Python 3.9+ and pulls `beancount` as a dependency,
so a single `pipx install fava` gives you both the web UI and the
underlying `bean-*` CLI tools.

## License

MIT — see
[LICENSE](https://github.com/beancount/fava/blob/main/LICENSE).
Permissive, retain the copyright notice in redistributions.
(Note that `fava`'s MIT license is independent of `beancount`'s
GPL-2.0 — you can ship MIT derivative works of `fava` itself,
but if you bundle the `beancount` library you inherit GPL
obligations on that piece.)

## One Concrete Example

Given the `book.beancount` from the [`beancount`](../beancount/)
entry:

```bash
# 1. start the local web app
fava book.beancount
# Running Fava 1.30.12 on http://127.0.0.1:5000

# 2. point an extra browser tab at a different sub-report directly
open "http://127.0.0.1:5000/example/income_statement/"
open "http://127.0.0.1:5000/example/balance_sheet/?time=2026-04"
open "http://127.0.0.1:5000/example/journal/?account=Assets:Checking"

# 3. run a Beancount-Query-Language query in the built-in editor
#    (also reachable as a URL, e.g. for bookmarking a recurring report)
open "http://127.0.0.1:5000/example/query/?query_string=SELECT+account%2C+sum%28position%29+WHERE+account+~+%27Expenses%27+GROUP+BY+account"

# 4. import a bank CSV — define an importer in ~/.config/fava/importers.py
#    then drop the CSV into the configured inbox; fava shows draft postings
#    you tick / edit / commit, and writes them into the .beancount file

# 5. live-reload — fava watches the source file; save book.beancount in
#    your editor and the open browser tab refreshes within ~1s

# 6. run as a background daemon (systemd / launchd unit)
fava --host 127.0.0.1 --port 5000 \
     --no-browser \
     book.beancount &

# 7. extension API — drop a Python module into the same dir and reference
#    it in the book:
#    2026-01-01 custom "fava-extension" "my_ext" "{'option': 'value'}"
#    fava picks it up on next reload and renders any custom routes /
#    sidebar entries it registers
```

## Niche It Fills

**A real GUI for plain-text accounting without giving up the text
file.** The trade-off most people make against `ledger` /
`beancount` is "I love the text file but I can't browse it the way
I could in QuickBooks". `fava` removes that trade-off: the file
remains the source of truth (still `git`-able, still grep-able,
still editable in `vim`), and the web app is a *view* on top — it
re-renders from the parsed AST on every load, and any edits it
writes back go through the same canonical formatter `bean-format`
uses. You get the GUI for browsing and the text file for
versioning, simultaneously.

## Why use it

1. **Zero-state web view of a typed book.** `fava book.beancount`
   in one terminal, browser tab in another, and you have a
   clickable account tree, balance sheet, P&L, cash-flow, and
   holdings view that match `bean-query` outputs exactly because
   they share the parser. There is no DB to migrate, no
   credential to rotate; killing the process loses nothing.
2. **Built-in BQL query editor with autocomplete.** The
   `query` page exposes the full Beancount Query Language with
   syntax highlighting, account autocomplete, and shareable
   query URLs. For ad-hoc "how much did I spend on X by month
   last year" investigations, this is faster than dropping to
   `bean-query` on the CLI, and the URL is bookmarkable as a
   saved report.
3. **Import workflow turns CSV into reviewed text.** Write a
   small Python `Importer` subclass per bank account; drop a CSV
   in the configured inbox; `fava` shows the proposed
   `Transaction` records side-by-side with their account
   guesses, you confirm or edit, and `fava` appends the typed
   text directly to the right `.beancount` file with the
   canonical formatting. The text-file invariant survives.

For an LLM-CLI workflow, `fava` is the human review surface that
complements `bean-check` (the agent edits the text file and runs
`bean-check`; the human opens `fava` to see the resulting balance
sheet move). The HTTP endpoints (`/account/...`, `/query/...`)
also return JSON when called with `Accept: application/json`,
making `fava` a thin REST shim over the parsed AST when an agent
needs structured snapshots.

## Vs Already Cataloged

- **Vs [`beancount`](../beancount/):** complementary, not
  competing. `beancount` is the parser + CLI tools (`bean-check`,
  `bean-query`, `bean-format`); `fava` is the web UI built on
  top of `beancount`'s typed AST. Most beancount users install
  both. `fava` is `pipx install fava`, which pulls `beancount`
  as a dep — there is no separate installation choice in
  practice.
- **Vs [`ledger`](../ledger/) and [`hledger`](../hledger/):**
  `fava` only speaks `.beancount`. The peer for `ledger` is
  `ledger-web` (third-party, lightly maintained); the peer for
  `hledger` is the bundled `hledger-web` (built-in, simpler
  feature set, no BQL). If you want the strongest GUI in the
  plain-text-accounting family, `fava` is the answer, and it
  pulls you to `beancount` as the underlying format.
- **Vs SaaS accounting (QuickBooks Online / Xero / YNAB):** the
  obvious gap is "no cloud sync, no mobile app, no bank-feed
  auto-import". `fava` runs on `127.0.0.1` against your local
  text file — that is the feature, not the bug. If you want
  cloud sync, use `git` + a private remote; if you want mobile,
  port-forward over Tailscale and use the browser. The
  trade-off you accept is "I run my own books on my own
  hardware".

## Caveats

- **Single-user by design.** `fava` has no auth layer beyond
  binding to `127.0.0.1`. The `--host 0.0.0.0` flag exists for
  LAN convenience, but exposing `fava` to the public internet
  without a reverse proxy doing auth (e.g. nginx + basic auth,
  or a Tailscale-only listener) means anyone who reaches the
  port reads — and, via the editor pane, writes — your book.
- **Editor pane writes back to the file.** Any change saved in
  the in-browser editor lands in `book.beancount` immediately;
  there is no per-edit confirmation. Always `git`-commit your
  book regularly, ideally on a `pre-`/`post-`save hook, so the
  diff is reviewable.
- **Live reload assumes a single writer.** If your editor and
  `fava`'s editor pane both have the same file open, the last
  save wins — there is no merge UI. Pick one writer per session,
  or rely on `git` for after-the-fact reconciliation.
- **Importers are user-written Python.** The CSV → draft
  transactions workflow is powerful but is not "fill in the bank
  URL". Each bank statement format needs a ~40-line Python
  `Importer` subclass; the [`beangulp`] and [`smart_importer`]
  community projects supply reference implementations for the
  common US banks but international coverage is uneven.
- **Browser-rendered UI inherits Svelte/JS deps.** `fava` ships
  with its frontend pre-built into the wheel, so end users do
  not need Node — but if you patch the UI, you need a Node
  toolchain (`make` + `npm` in the repo). Pure-CLI ops never
  trip this; only contributors do.
