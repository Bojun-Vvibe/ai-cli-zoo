# cicada

> **A bash-like Unix shell written in Rust** — small, fast, and
> opinionated: built-ins for `cd`, `alias`, `export`, `vox`,
> globbing, brace expansion, command substitution, `$( )`,
> backticks, here-strings, redirections, pipes, and job control;
> SQLite-backed history with fuzzy search; per-directory env
> hooks; a single ~3 MB static binary that drops into `/etc/shells`.
> Pinned to **v1.0.4**
> ([LICENSE](https://github.com/mitnk/cicada/blob/master/LICENSE),
> MIT).

Source: <https://github.com/mitnk/cicada>

## TL;DR

`cicada` is what you reach for when you want a Rust-clean
interactive shell that still feels like bash to your fingers
but does not inherit four decades of POSIX warts. The grammar
is intentionally a strict subset — pipes, redirections,
`&&` / `||`, `;`, command substitution, glob, brace expansion,
single/double-quoted strings, here-strings — and stops there:
no `[[ ... ]]`, no associative arrays, no process substitution,
no `function f() { ... }` declarations. Scripts run via shebang
(`#!/usr/bin/env cicada`), but the design center is the
interactive REPL: SQLite history at `~/.local/share/cicada/history.sqlite`
that survives across sessions and machines (sync the file), a
built-in `vox` virtualenv manager, regex-driven `alias` rules,
per-directory `.cicada_env` auto-load, and a prompt DSL with
git-aware segments.

The trade is honest: you give up bash script compatibility (no
`bash -c '...'` semantics in `.sh` files you source) in exchange
for a shell that starts in <10 ms cold, never segfaults, and
whose entire grammar fits in your head in an afternoon. Pair it
with `starship` (works as-is — cicada honors `STARSHIP_SHELL=bash`)
and you have a daily driver.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install cicada

# Cargo (any platform)
cargo install -f cicada

# from source
git clone https://github.com/mitnk/cicada && cd cicada
cargo build --release
sudo cp target/release/cicada /usr/local/bin/

# add to /etc/shells and chsh
echo "$(command -v cicada)" | sudo tee -a /etc/shells
chsh -s "$(command -v cicada)"

# verify
cicada --version    # 1.0.4
```

## Examples

```bash
# interactive — looks like bash, behaves like bash for the 80%
$ cicada
(in) ~ $ ls *.rs | head -3
(in) ~ $ echo "today is $(date +%F)"
(in) ~ $ cd ~/code && git status

# pipes + redirections + && / ||
(in) ~ $ cargo build --release 2>&1 | tee build.log && say done

# brace + glob expansion
(in) ~ $ mkdir -p src/{api,cli,core}/tests
(in) ~ $ rm -f *.{tmp,bak,swp}

# SQLite history fuzzy recall (Ctrl-R)
(in) ~ $ <Ctrl-R>cargo bui<Enter>     # replays last `cargo build --release`

# per-directory env hooks (.cicada_env in $PWD)
(in) ~/proj $ cat .cicada_env
export DATABASE_URL="postgres://localhost/dev"
export RUST_LOG=debug
# auto-loaded on `cd` into the dir, unloaded on exit

# vox: built-in Python virtualenv manager
(in) ~ $ vox create scratch
(in) ~ $ vox enter scratch
(in) [scratch] ~ $ pip install requests

# alias with regex capture
(in) ~ $ alias 'gpr (\d+)' 'gh pr view $1 --web'
(in) ~ $ gpr 482     # opens PR #482 in browser
```

## Why it matters

Most "modern shell" projects (`fish`, `nushell`, `elvish`,
`oils`, `xonsh`) reinvent the language. Cicada is the rare
counter-bet: keep bash-shaped syntax for the 80% of muscle
memory that matters, drop the corners that cause CVEs and
shellcheck warnings, rewrite the runtime in Rust, ship one
static binary, and add the two QoL features every shell user
actually wants — searchable persistent history and per-dir env
auto-load — as built-ins instead of plugins.

## Comparison

| Shell      | Bash-compat REPL | Static binary | Persistent fuzzy history | Per-dir env | Lang reinvention |
| ---------- | ---------------- | ------------- | ------------------------ | ----------- | ---------------- |
| `cicada`   | high (subset)    | yes (~3 MB)   | yes (sqlite, built-in)   | built-in    | no               |
| `bash`     | n/a              | no            | flat-file, no fuzzy      | via direnv  | n/a              |
| `fish`     | low              | no            | yes (history search)     | via plugin  | yes              |
| `nushell`  | none             | yes           | yes (sqlite)             | env config  | yes (data lang)  |
| `elvish`   | low              | yes           | yes                      | yes         | yes              |
| `oils`     | very high (OSH)  | no            | flat-file                | no          | partial (YSH)    |

## License

- License: **MIT**
- Path in repo: `LICENSE`
- URL: <https://github.com/mitnk/cicada/blob/master/LICENSE>
