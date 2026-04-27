# watchexec

> **Run arbitrary commands when files change.** Pinned to
> **v2.5.1**, Apache-2.0
> ([LICENSE](https://github.com/watchexec/watchexec/blob/main/LICENSE)).

- **Repo:** https://github.com/watchexec/watchexec
- **Latest version:** v2.5.1
- **License:** Apache-2.0 (`LICENSE` at repo root, SPDX `Apache-2.0`)
- **Category:** `dev-experience` / `automation`
- **Language:** Rust

## What it does

`watchexec` is a single static binary that watches a directory tree
for filesystem events (using the native `notify` crate — `FSEvents`
on macOS, `inotify` on Linux, `ReadDirectoryChangesW` on Windows)
and runs a shell command in response. Out of the box it understands
`.gitignore` / `.ignore` files so build artefacts and `node_modules`
are skipped, debounces bursts of events, kills the previous run when
a new event arrives (`--restart`), and supports per-extension
filtering (`-e rs,toml`), inverse globs, env-var passthrough of the
changed paths (`$WATCHEXEC_WRITTEN_PATH`), and clearing the screen
between runs (`-c`). Typical invocations: `watchexec -e rs cargo
test`, `watchexec -r -- cargo run`, `watchexec -w src -- make`.
The same engine powers the file-watching loops inside `cargo-watch`
and several other ecosystem tools.

## Why included

The canonical "rerun X when files change" primitive — language
agnostic, no config file needed, sane defaults around ignore files
and process lifecycle. Replaces ad-hoc `entr` / `nodemon` /
`reflex` / `fswatch` shell pipelines with one binary that behaves
the same on every OS. Pairs well with [`hyperfine`](../hyperfine/)
(benchmark on save) and [`just`](../just/) (`watchexec just test`).
