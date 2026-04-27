# bacon

> **Background Rust code-checker that lives in a side pane and re-runs
> `cargo check` / `clippy` / `test` / `nextest` / `doc` on every save,
> rendering a one-screen distilled error view.** Pinned to **v3.22.0**,
> AGPL-3.0
> ([LICENSE](https://github.com/Canop/bacon/blob/main/LICENSE)).

- **Repo:** https://github.com/Canop/bacon
- **Latest version:** v3.22.0
- **License:** AGPL-3.0 (`LICENSE` at repo root, SPDX `AGPL-3.0`)
- **Category:** `dev-experience` / `rust-tooling`
- **Language:** Rust

## What it does

`bacon` watches the file tree of a Rust project and on every save
runs a configured `cargo` job — `check` (default, fastest), `clippy`,
`clippy-all` (`--all-targets --all-features`), `test`,
`nextest`, `doc`, `run`, or any custom job declared in
`bacon.toml` — then renders the result as a single-screen distilled
view: warnings collapsed by default, errors expanded with file:line
links, build-stage progress at the bottom, and `c` / `w` / `t` / `d`
keystrokes hot-swap between jobs without restart. Sits in a small
multiplexer pane (`tmux` / `zellij` / `kitty` split) so the editor
stays focused on code while the compiler answers in the background.
Per-project `bacon.toml` (or `.bacon-locations.toml`) declares custom
jobs (`cross check`, `cargo +nightly clippy`, `cargo nextest run -E
'package(my-crate)'`), keybindings to invoke them, and on-success
hooks (`on_change_strategy = "kill_then_restart"` for long jobs).
The `--headless` mode emits structured output for CI / agents.

## Why included

The "save → 8 s `cargo check` round-trip" loop is the biggest single
cost of writing Rust on a laptop, and the standard fix
(`cargo-watch`) re-prints raw compiler output that scrolls off-screen
the moment a second error fires. `bacon` keeps the *current* error
set on one screen, deduplicated and sorted by severity, so a
multi-file refactor's 40-warning storm is readable instead of a
firehose. The hot-swap keybindings make "is this clippy-clean and
does the test still pass?" a two-keystroke check (`c` → `w` → `t`)
without re-typing `cargo` invocations. For an LLM-CLI workflow that
asks an agent to refactor Rust, `bacon` running in a sibling pane is
the human-side feedback loop that catches "agent's diff compiles but
trips clippy" before the next agent turn.
