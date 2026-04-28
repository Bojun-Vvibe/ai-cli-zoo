# ov

> **Feature-rich terminal-based text viewer / pager** — a `less(1)`
> alternative in Go with native column / CSV awareness, multi-file
> tabs, follow-mode that survives log rotation, and a built-in
> filter-as-you-type pipeline. Pinned to **v0.52.0** (commit
> `ef04031cc6aa6dda4eab88f1b8a6f394e8b16a22`,
> [LICENSE](https://github.com/noborus/ov/blob/master/LICENSE),
> MIT).

Source: <https://github.com/noborus/ov>

## TL;DR

`ov` is the pager you reach for when `less` gives up. It opens any
file (or stdin), but it also understands **structure**: pass `--mode csv`
or `--mode tsv` and it renders the file as a column-aligned table with
sortable, filterable columns; pass `-H 1` to freeze a header row that
stays put while you scroll; pass `--section-delimiter '^---'` to teach
the pager about logical sections (markdown documents, multi-table SQL
output, long Ansible task logs) and use `n` / `p` to jump between them.
**Follow-mode** (`F`, like `less +F` or `tail -f`) handles log rotation
correctly — when the inode changes, `ov` re-opens the path, so it does
not silently stop tailing your nginx logs at midnight. There is a
**filter prompt** (`&`) that hides every line not matching a regex
(toggleable, stackable), a **search prompt** (`/`) that highlights
without filtering, **multi-file mode** with `Ctrl-N` / `Ctrl-P` to flip
between files, and a **plain-mode escape hatch** (`-p`) when you just
want `less` semantics with a faster Go binary.

## Install

```bash
# Homebrew (macOS / Linux)
brew install noborus/tap/ov

# Go
go install github.com/noborus/ov@latest

# Replace less for the current shell
export PAGER=ov
```

```bash
# CSV with frozen header and column sort
ov --mode csv -H 1 sales.csv

# Follow a log that rotates
ov -F /var/log/nginx/access.log

# Stream stdin from psql with column alignment
psql -c "select * from users" | ov --mode psql
```

## Niche

The "**structure-aware pager**" slot. Where
[`bat`](../bat/) is a syntax-highlighting `cat` (no interactive
viewport, no follow-mode), and where `less` is the universal lowest-
common-denominator pager (no column awareness, follow-mode breaks on
rotation, no per-format rendering modes), `ov` sits in the middle: a
proper interactive pager that *also* knows the difference between
"rendering 80 GB of newline-delimited JSON" and "rendering a 200-row
CSV with a header". The format-mode set covers CSV, TSV, PSQL output,
markdown, and a generic regex-delimited section mode, so the same
binary reads `kubectl get pods -o wide`, your migration log, and a
data dump without you having to pick a different tool per format.

## Why it matters

- **Format-aware modes** — `--mode csv|tsv|psql|markdown|...` reflows
  the input as a column-aligned table that you can scroll horizontally
  with frozen headers (`-H N`) and frozen leading columns (`-y N`).
  `less` cannot do this; you would shell out to `column` and lose the
  pager.
- **Follow-mode that survives rotation** — `F` opens the file by path,
  not by fd, and re-opens on inode change. This is the single most
  common reason teams stop trusting `less +F` on production logs.
- **Filter + search are separate prompts** — `/regex` highlights
  in-place; `&regex` hides non-matching lines; both are toggleable and
  stack with the format-mode renderer, so you can filter a CSV by a
  column-value regex without losing the column alignment.
- **MIT, single Go binary, no daemon** — drop-in `PAGER=ov`
  replacement; falls back to plain-text mode (`-p`) when you want
  identical-to-`less` behaviour for muscle memory.
