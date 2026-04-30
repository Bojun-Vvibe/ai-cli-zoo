# ttyper

- **Repo:** https://github.com/max-niederman/ttyper
- **Latest version:** 1.6.0
- **License:** MIT (`LICENSE.md`)
- **Category:** dev-tools / typing-test

A terminal-based typing test written in Rust, rendered with `tui-rs`-style
widgets, that drills you on real word lists or arbitrary text files and
shows live WPM, accuracy, and per-key heatmaps when the run finishes.
Choose ttyper over web typing tests when you want to stay in the terminal
between coding sessions, when you need an offline drill that doesn't ship
your keystrokes to a SaaS, or when you want to feed it a custom corpus
(your own commit messages, a language's keyword list, regex syntax) to
practice the characters you actually type. Single static binary, no
network, no telemetry.

## Install

```bash
# Cargo
cargo install ttyper

# Homebrew
brew install ttyper

# Arch
pacman -S ttyper

# verify
ttyper --version
```
