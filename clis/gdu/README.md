# gdu

- **Repo:** https://github.com/dundee/gdu
- **Version:** v5.36.0 (latest stable, 2026-04-27)
- **License:** MIT ([LICENSE.md](https://github.com/dundee/gdu/blob/master/LICENSE.md))
- **Language:** Go
- **Install:** `brew install gdu` · `go install github.com/dundee/gdu/v5/cmd/gdu@latest` · `cargo binstall gdu` (via mise/binstall) · `pacman -S gdu` (Arch) · `apt install gdu` (Debian/Ubuntu via the upstream `.deb` on the release page) · static release binaries on the GitHub release page for Linux / macOS / FreeBSD / Android (amd64, arm64, 386, arm)

## What it does

`gdu` is a **fast disk-usage analyzer with an interactive TUI**, built
to exploit SSD parallelism — it walks the filesystem with a goroutine
pool sized to your CPU and finishes a `~/` scan in seconds where
classic `du -sh *` would single-thread for minutes. The default mode
opens a navigable two-pane TUI (directory list left, sizes + percent
bars right) with vim-style keybinds for descending / ascending,
deletion, sort cycling, and an `--archive-browsing` flag that treats
zip / jar / tar archives as directories so you can see what is actually
eating space inside a fat artifact.

## Minimal usage example

```bash
# 1. Open the interactive TUI on the current directory (most common)
$ gdu
# expected: full-screen TUI; top status line shows total size + item
# count being analysed live; once done, directories sorted by size with
# a percent bar; arrow keys / hjkl navigate, `d` deletes (confirmation
# popup with default = no), `a` toggles apparent vs disk usage,
# `n/s/c/M` cycle sort by name/size/items/mtime, `q` quits

# 2. Scan a specific path (typical "what is filling up /var" reflex)
$ sudo gdu /var
# expected: same TUI but rooted at /var; useful with sudo on Linux to
# count files your user cannot stat

# 3. Non-interactive top-N report for scripts / CI / cron (v5.30.0+)
$ gdu --non-interactive --top 20 /home
# expected: a plain-text table of the 20 largest files anywhere under
# /home, one per line, size + path, no TUI, exits 0; safe to pipe into
# `mail` or a Slack webhook from a nightly job

# 4. Print a JSON-able analysis of a path with no UI at all
$ gdu --non-interactive --no-progress --show-disk-usage /tmp
# expected: one line per top-level entry in /tmp with "size  itemcount  path"
# columns, no ANSI, suitable for `awk` / `jq` post-processing

# 5. Find big stuff while excluding the usual VCS / cache noise
$ gdu --ignore-dirs '/proc,/sys,/dev,/run' --no-cross /
# expected: full-disk scan that skips kernel pseudo-fs and refuses to
# cross into other mounts, so the TUI shows only what is on the root fs
```

## When to pick it / when not to

Pick `gdu` when you need to **find what is using space on a real SSD,
fast** — the parallel walk genuinely uses an NVMe drive's queue depth
where `du`, `ncdu`, and even `dust` either single-thread or use much
lower parallelism, so on a 2 TB working drive the difference is
"finished while you tab back to the browser" vs "still running when you
come back from a coffee". Pair with [`dust`](../dust/) (dust is the
quick "one shot, pretty bar chart, no TUI" answer; gdu is the "I want
to navigate and delete" answer), with [`duf`](../duf/) (duf shows
*which* filesystems are full, gdu shows *what inside one* is full —
they answer adjacent questions), with [`fclones`](../fclones/) /
[`kondo`](../kondo/) (once gdu surfaces the offenders, fclones
deduplicates and kondo wipes language-ecosystem build caches), and with
[`broot`](../broot/) (broot is a fuzzy navigator with size hints; gdu
is purpose-built for size triage with deletion).

Skip it on spinning rust — the parallel walk causes seek thrashing on
HDDs and `ncdu`'s sequential scan ends up faster. Skip it for
"summarise one path" one-shots — `du -sh path` is two characters and
you already have it. Skip it for network filesystems where the bottleneck
is the wire, not the local CPU. Skip it on Windows — Linux / macOS /
FreeBSD / Android are first-class, Windows is community-best-effort
and the TUI is the macOS / Linux experience.
