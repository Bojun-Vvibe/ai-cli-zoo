# rink

- **Repo:** https://github.com/tiffany352/rink-rs
- **Latest version:** 0.9.0
- **License:** GPL-3.0 (`LICENSE-GPL`; library portion under MPL-2.0 via `LICENSE-MPL`)
- **Category:** dev-tools / calculator

A units-aware calculator and conversion tool, written in Rust, with a REPL
that knows about thousands of physical units, currencies, and dimensional
analysis rules out of the box. Type `8 GiB / 50 Mbps -> minutes` and rink
returns a properly-dimensioned answer; type `kWh -> J` and it converts.
Reach for it instead of `qalc` or `units(1)` when you want a single static
binary with offline currency data, a forgiving expression syntax that
catches dimension mismatches at parse time, and one-shot mode for shell
pipelines (`rink -1 "c in mph"`). Useful in CLI workflows that need quick
back-of-envelope math without spinning up Python or a browser tab.

## Install

```bash
# Cargo
cargo install rink

# Homebrew
brew install rink

# verify
rink --version
```
