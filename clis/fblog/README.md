# fblog

> **Small command-line JSON log viewer** — pretty-prints
> structured (JSON / logfmt) log streams into human-readable
> rows with colourised levels, selectable fields, and filtering
> via embedded Lua expressions. Reads stdin or files, tail-
> friendly. Pinned to **v4.13.1** (SPDX: `MIT`,
> [LICENSE](https://github.com/brocode/fblog/blob/master/LICENSE)).

Source: <https://github.com/brocode/fblog>

## TL;DR

`fblog` is what you pipe `kubectl logs` / `journalctl -o json` /
`docker logs` / a structured app log into when the raw JSON is
unreadable but `jq` is too programmatic. It auto-detects common
log shapes (Bunyan, Logrus, plain JSON, logfmt), highlights the
level, prints a compact `time level message` line by default,
and lets you add/remove visible fields with `-a request_id`
or filter with a one-liner Lua predicate (`-f 'level == "error"'`).

## Install

```bash
# Homebrew (macOS / Linux)
brew install fblog

# Cargo
cargo install --locked fblog

# Arch
pacman -S fblog

# Pre-built binary
curl -Lo fblog "https://github.com/brocode/fblog/releases/download/v4.13.1/fblog-aarch64-apple-darwin"
chmod +x fblog && sudo install fblog /usr/local/bin/

# verify
fblog --version    # fblog 4.13.1
```

## License

MIT — see
[LICENSE](https://github.com/brocode/fblog/blob/master/LICENSE).

## One Concrete Example

```bash
# 1. pretty-print a JSON log file
fblog app.log

# 2. tail a structured log stream
tail -f app.log | fblog

# 3. add extra fields to the visible row (default shows only
#    time / level / message)
tail -f app.log | fblog -a request_id -a user_id

# 4. show ONLY specific fields (override the defaults)
fblog -p time -p level -p msg -p err app.log

# 5. filter with a Lua predicate (only errors)
fblog -f 'level == "error" or level == "fatal"' app.log

# 6. filter by substring in any field
fblog -f 'string.find(message, "timeout") ~= nil' app.log

# 7. dump everything (no level filtering, every field)
fblog -d app.log

# 8. parse logfmt instead of JSON
fblog --logfmt app.log

# 9. point at a non-standard message field (e.g. "msg" -> "log")
fblog -m log -t timestamp app.log
```

## Niche It Fills

**Reading structured logs with eyes, not jq programs.**
`jq` excels at *projecting* / *transforming* JSON; `fblog` excels
at the simpler upstream task of *reading* a stream of structured
log lines as if they were `tail -f` over plain text. The Lua
filter handles 90 % of the "show me only the bad lines" case
without writing a `jq` `select` expression.

## Why use it

1. **Auto-detects common log shapes.** Bunyan, Logrus, generic
   JSON, logfmt — works on most Go / Node / Rust app logs out of
   the box without configuration.
2. **`-a` adds fields, `-p` replaces them.** Two-axis control
   over verbosity that `jq` requires a custom expression for
   each time.
3. **Lua filter language.** `-f 'level == "error" and
   string.find(path, "/api/")' ` is more readable than the `jq`
   equivalent and embeds in shell history cleanly.
4. **Streams.** Works on `tail -f` and `kubectl logs -f`
   without buffering surprises.
5. **Tiny binary, no runtime.** Single Rust binary, no Node /
   Python / JVM.

## Vs Already Cataloged

- **Vs [`jq`](../jq/):** complementary — `jq` is the right
  answer when you want to *extract* / *reshape* JSON values
  into another pipeline; `fblog` is the right answer when you
  want to *read* a structured log stream as a human. Pair:
  `kubectl logs ... | jq -c 'select(.svc=="api")' | fblog`.
- **Vs [`lnav`](../lnav/):** `lnav` is a full TUI with SQL
  over your logs, format auto-detection, and persistent
  indices. Heavier and more capable. Reach for `lnav` for an
  investigation; reach for `fblog` for a one-shot pipeline.
- **Vs [`hl`](../hl/) / [`humanlog`](../humanlog/):** same
  niche (pretty-print structured logs). `fblog` differentiates
  on the embedded Lua filter; `humanlog` on broader format
  auto-detection. Try both — they cost nothing to install side
  by side.

## Caveats

- **One-line JSON only by default.** Multi-line JSON (pretty-
  printed) confuses the line-by-line parser. Re-encode with
  `jq -c .` first.
- **Lua filter is per-line and synchronous.** Heavy `string.find`
  on huge volumes is slower than a `grep` pre-filter. For
  multi-GB files: `grep ERROR raw.log | fblog -f '...'`.
- **Not a parser for unstructured logs.** If your app emits
  plain `[2026-04-29 10:00:00] INFO doing thing`, `fblog` will
  show each line as a single message field with no level
  colouring. Use `lnav` for that.
- **Field-name conventions vary.** If your log uses `severity`
  / `lvl` / `log_level` instead of `level`, pass `-l severity`.
  Same for `-t` (time) and `-m` (message).
