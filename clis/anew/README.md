# anew

> **A tool for adding new lines to files, skipping duplicates** —
> a tiny Go binary by tomnomnom that reads stdin, appends only
> the lines that don't already appear in a target file, and
> echoes the newly-added lines to stdout. Acts like
> `tee -a` minus the duplicates. Pinned to **v0.1.1**
> ([LICENSE](https://github.com/tomnomnom/anew/blob/master/LICENSE),
> MIT).

Source: <https://github.com/tomnomnom/anew>

## TL;DR

`anew` solves one tiny but constantly-recurring problem: you
have a streaming source of lines (a crawl, a log scrape, a
nightly export, a `find` over a watched tree) and you want to
maintain a *cumulative, deduplicated* file of everything seen so
far — and you want to know, on each run, *which lines were
new*. The single-file Go program does exactly that:

- reads stdin one line at a time;
- compares each line against the in-memory set of lines already
  in the target file;
- if it's new, appends it to the file *and* writes it to stdout;
- if `-d` (dry-run), only writes new lines to stdout;
- if `-q` (quiet), only appends, no stdout output.

Because new lines come out on stdout, you can chain `anew` into
the next stage of a pipeline and get incremental processing
"for free" — the next stage only ever sees first-time records.

## Install

```bash
# Go (recommended)
go install -v github.com/tomnomnom/anew@latest

# Homebrew (macOS / Linux)
brew install anew

# Pre-built binary
curl -LO https://github.com/tomnomnom/anew/releases/download/v0.1.1/anew-linux-amd64-0.1.1.tgz
tar xf anew-linux-amd64-0.1.1.tgz
sudo install anew /usr/local/bin/

# Verify
anew --help    # prints usage; v0.1.1 is the current release
```

## Example usage

```bash
# 1. Maintain a cumulative URL list; print only first-time URLs
cat new-urls.txt | anew all-urls.txt
# stdout: only URLs not already in all-urls.txt
# all-urls.txt: now contains the union

# 2. Dry-run — preview what would be added without touching the file
cat new-urls.txt | anew -d all-urls.txt

# 3. Quiet — append silently, suppress stdout
cat new-urls.txt | anew -q all-urls.txt

# 4. Capture only the diff to a separate file
cat new-urls.txt | anew all-urls.txt > added-this-run.txt

# 5. Incremental pipeline: only process *new* records downstream
crawl example.com \
  | anew seen-urls.txt \
  | xargs -I{} curl -sI {} \
  | tee fresh-headers.log

# 6. Daily cron: append today's findings, ship the diff to Slack
nightly-scan.sh \
  | anew /var/lib/scan/seen.txt \
  | mail -s "new findings $(date +%F)" team@example.com

# 7. Combine with sort/uniq for stable, sorted, dedup'd cumulative state
cat fresh.txt | anew -q seen.txt && sort -o seen.txt seen.txt
```

## Why this lives in the zoo

`anew` is the "missing primitive" between `tee -a` (appends
everything, including duplicates) and `sort -u` (deduplicates
but loses order and forces re-reading the whole file). It's
~80 lines of Go, has no config, no flags beyond `-d` / `-q`,
and composes cleanly with every other line-oriented Unix tool.
Once it's on your `PATH` you'll find yourself reaching for it
any time a script needs the answer to "what's new since last
time?" — incremental crawls, append-only changelogs, watching a
log for previously-unseen error signatures, building a corpus
from streamed inputs. The kind of tiny tool that earns its
keep on the first day and stays for years.
