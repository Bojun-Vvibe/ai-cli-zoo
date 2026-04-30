# halp

- **Repo:** https://github.com/orhun/halp
- **Latest version:** v0.2.0
- **License:** Apache-2.0 (`LICENSE-APACHE`) / MIT (`LICENSE-MIT`)
- **Category:** dev-tools / cli-helper

A CLI tool that helps you find the help — when a command's `--help` is
absent, cryptic, or just not where you expected it. `halp` inspects an
arbitrary binary and tries every reasonable invocation (`-h`, `--help`,
`help`, `-?`, no args, man page lookup, version probes) in parallel,
then surfaces whichever one actually returned useful output. It also
falls back to `cheat.sh` for crowd-sourced examples when the binary
itself stays silent. Useful for old C utilities that only have a man
page, Go tools that print usage on stderr at exit code 2, or random
shell scripts you inherited and have no docs for. Pure Rust, single
static binary, no daemon.

## Install

```bash
# Cargo
cargo install halp

# Homebrew
brew install halp

# Arch (extra)
pacman -S halp

# verify
halp --version
```

## Why it's interesting

It's a meta-CLI: the tool whose entire job is to make every *other*
CLI on your system more discoverable. Solves the universal "I know
this binary does what I need, I just can't remember the flag" problem
without forcing you to leave the terminal for a browser tab.
