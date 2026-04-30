# so

- **Repo:** https://github.com/samtay/so
- **Latest version:** v0.4.10
- **License:** MIT (`LICENSE`)
- **Category:** dev-tools / search-tui

A terminal interface for Stack Exchange — search Stack Overflow (or
any other Stack Exchange site: superuser, serverfault, askubuntu,
unix.stackexchange, etc.) and read the top answers in a split-pane
TUI without leaving the terminal. Type a query, browse matched
questions in the left pane, render the accepted/highest-voted answer
(with code blocks, syntax highlighting, and markdown) in the right.
Vim-style keybindings, configurable site list, optional `--lucky`
mode that prints just the top answer to stdout (great for piping
into your editor or a script). Written in Rust.

## Install

```bash
# Cargo
cargo install so

# Homebrew
brew install so

# verify
so --version
```

## Why it's interesting

The `--lucky` flag turns Stack Overflow into a unix-philosophy
component: `so --lucky "how to grep recursively excluding dir"`
prints the answer body to stdout, ready to pipe into `bat`, `glow`,
or your clipboard. It's the missing CLI for the part of dev work
that's still "switch to browser, search, copy code, switch back" —
collapsed into one command in the same shell where you'll use the
result.
