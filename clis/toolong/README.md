# toolong

> **A Textual TUI for tailing, searching, and merging log files** —
> a Python single-entry tool (`tl`) that opens log files (including
> `.gz`) in a fast scrollable viewer with regex search, JSON-lines
> pretty-printing, live tail, and timestamp-merged multi-file view.
> Pinned to **v1.4.0**
> ([LICENSE](https://github.com/Textualize/toolong/blob/main/LICENSE),
> MIT).

Source: <https://github.com/Textualize/toolong>

## Category

Log viewer / TUI. Sister tool to `lnav` but built on
[Textual](https://github.com/Textualize/textual) — lighter
surface, prettier defaults, zero-config.

## What it does

Run `tl file.log` and you get a Textual TUI: arrow-key scroll,
`/` for regex search, `f` to toggle live tail, `m` to merge
multiple files into one timestamp-sorted pane, automatic
JSON-lines pretty-printing on the focused line, and transparent
gzip support so `tl access.log.gz` just works. Pipe-friendly:
`cat foo.log | tl` opens the pager directly on the pipe.

## Why it matters

Log viewing on a laptop usually devolves into `less +F` plus a
mental model of "I'll grep when I find something interesting."
`tl` replaces that with a real scrollable surface that handles
massive files (it indexes lazily, so 10 GB opens instantly),
auto-pretty-prints JSON log lines as you scroll past them, and
merges multiple files on timestamp without any config. Zero
flags, zero format spec — just point it at the files.

## Install

```bash
# pipx (recommended — isolated venv)
pipx install toolong

# pip
pip install --user toolong

# uv tool
uv tool install toolong

# Verify
tl --version    # toolong, version 1.4.0
```

## License

MIT — see
[LICENSE](https://github.com/Textualize/toolong/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# open a single file (works on huge files, indexes lazily)
tl /var/log/syslog

# open a gzipped log directly — no need to gunzip first
tl access.log.2025-04-01.gz

# merge N files into one timestamp-sorted view
tl app/*.log

# tail a pipe — same TUI, with live updates
kubectl logs -f deploy/api | tl

# inside the TUI:
#   /         regex search forward
#   ?         regex search backward
#   f         toggle live tail
#   m         toggle merged view across files
#   PgUp/PgDn page through
#   q         quit
```

## Niche It Fills

`less +F` is universal but ugly and JSON-blind. `lnav` is more
powerful but has a steep config surface. `tl` is the
"open-it-and-it-just-works" middle ground: zero config, gzip
transparent, JSON-aware, multi-file merge by default, and a
Textual TUI that looks good enough to share in a screenshot. The
killer feature is opening a 10 GB file or a live `kubectl logs`
pipe with no warm-up time and getting search + tail in one
keystroke each.
