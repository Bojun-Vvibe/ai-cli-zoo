# cargo-watch

> **Re-run Cargo commands when your source changes** — a tiny Rust
> file-watcher that wraps `cargo check` / `cargo test` / `cargo run`
> (or any shell command) in a debounced filesystem-event loop, so
> editing a `.rs` file instantly reruns whatever feedback command
> you care about. The de facto "save and see results" loop for Rust
> development for the better part of a decade. Pinned to **v8.5.3**
> (commit `58e792b99fade3b0c0ad198fd9627dcd4817eb37`,
> [LICENSE](https://github.com/watchexec/cargo-watch/blob/v8.5.3/LICENSE),
> CC0-1.0 public-domain dedication).

Source: <https://github.com/watchexec/cargo-watch>

## TL;DR

`cargo-watch` installs as a Cargo subcommand (`cargo watch`) and
turns Cargo into a live-reload loop. By default it watches the
package source tree (it ignores `target/`, `.git/`, hidden files,
and anything in `.gitignore`), debounces filesystem events for
500 ms so a multi-file save fires once, and reruns `cargo check`
on each change — meaning you get type errors and clippy lints in
under a second of feedback latency without leaving your editor.
Pass `-x <cargo subcommand>` to chain whatever you actually want
(`-x check -x test -x 'run -- --port 8080'` runs all three in
order, stopping early on failure); pass `-s <shell command>` to
run arbitrary non-Cargo commands (`-s 'cargo fmt && cargo clippy'`).
Useful flags: `-c` clears the screen between runs, `-q` silences
cargo-watch's own output, `-i 'docs/**'` excludes globs beyond
`.gitignore`, `-w src/` narrows the watched set, `-d 2` overrides
the debounce, `-N` disables `notify`-based watching in favour of
polling for finicky network filesystems. Internally it shells
out to the same `cargo` binary on your `$PATH`, so it picks up
your usual `Cargo.toml`, workspace, target dir, and
`RUSTFLAGS` without configuration. Note: the project is now in
maintenance mode and the upstream README points users at the more
general-purpose `watchexec` / `cargo binstall watchexec-cli` for
new projects, but `cargo-watch` remains the muscle-memory binding
in countless `Justfile`s and `Makefile`s.

## Install

```bash
# from crates.io (single static binary, no runtime deps)
cargo install cargo-watch --version 8.5.3 --locked

# Homebrew
brew install cargo-watch

# pre-built binaries (x86_64 / aarch64 Linux + macOS + Windows)
cargo binstall cargo-watch

# verify
cargo watch --version     # cargo-watch 8.5.3
```

## Examples

```bash
# 1. The classic "save = type-check" loop while editing a library crate
cargo watch -c -x check
#   -c clears the screen each iteration, -x runs `cargo check`

# 2. Test-driven loop for a binary: format on save, run a focused test,
#    then restart the binary with arguments — stop the chain on first failure
cargo watch -c \
  -s 'cargo fmt' \
  -x 'test --lib parser::' \
  -x 'run -- serve --port 8080' \
  -i 'fixtures/**' -i '**/*.snap.new'
```
