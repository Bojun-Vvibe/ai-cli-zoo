# shfmt

> **A formatter for shell scripts** — parses
> POSIX `sh`, `bash`, and `mksh` into an AST and
> rewrites it with consistent indentation,
> spacing, and redirection style. Pinned to
> **v3.13.1**
> ([LICENSE](https://github.com/mvdan/sh/blob/master/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/mvdan/sh>

## TL;DR

`shfmt` is the "gofmt for shell" tool. It ships
inside Daniel Martí's `mvdan/sh` repo, which is a
pure-Go shell parser/printer/interpreter. The
`shfmt` binary takes any `sh` / `bash` / `mksh`
script and rewrites it to a canonical form:
configurable indent width (`-i 2`), case-branch
indentation (`-ci`), space after redirection
(`-sr`), function-opener style (`-fn`), and a
`-bn` flag that puts binary operators (`&&`, `|`)
at the start of the next line. It can format in
place (`-w`), diff (`-d`), or simply check that
files are already formatted (`-l`), which makes
it a perfect pre-commit / CI gate. Because it is
a real parser, not regex, it correctly handles
heredocs, here-strings, process substitution,
`$(( ... ))`, `[[ ... ]]`, and bash arrays — it
will refuse to format a file with a syntax error
rather than mangle it.

## Install

```bash
# Homebrew
brew install shfmt

# Go
go install mvdan.cc/sh/v3/cmd/shfmt@latest

# prebuilt binaries
# https://github.com/mvdan/sh/releases

# verify
shfmt --version    # v3.13.1
```

## Basic usage

```bash
# format in place, 2-space indent, indent case branches
shfmt -w -i 2 -ci script.sh

# show a diff without writing
shfmt -d ./scripts

# CI gate: exit 1 if anything would change
shfmt -d -i 2 -ci .

# only list files that need formatting
shfmt -l .

# read from stdin, write to stdout
cat install.sh | shfmt -i 4 -ci > install.formatted.sh

# format inside Markdown code fences (docs)
shfmt -w README.md
```

## When to choose

- **You maintain more than ~5 shell scripts**
  and want them to look the same — manual style
  policing in PRs is exhausting; let shfmt do it.
- **You already run [`shellcheck`](../shellcheck/)** —
  shfmt is the natural format-side companion;
  shellcheck catches bugs, shfmt enforces style.
- **You want a reliable pre-commit hook** —
  single static Go binary, fast on hundreds of
  scripts, supports check / write / diff modes.

## When NOT to choose

- **You only write Fish or PowerShell** — shfmt
  parses POSIX-family shells (`sh` / `bash` /
  `mksh`) only.
- **You want lint / static analysis** — use
  [`shellcheck`](../shellcheck/); shfmt rewrites
  layout but does not flag semantic issues.
- **You prefer hand-tuned alignment** — shfmt is
  opinionated about column position of
  comments/redirections; teams that align things
  visually will find that frustrating.

## Why it fits the zoo

Slots into the "language-aware formatter"
cluster alongside [`biome`](../biome/) (JS/TS),
[`stylua`](../stylua/) (Lua),
[`buf`](../buf/) (protobuf), and
[`taplo`](../taplo/) (TOML). shfmt's specific
gap is **AST-based formatting for POSIX shells**
— no other tool in the zoo formats `bash`
without falling back to regex heuristics, and
none ship a parser robust enough to format
heredocs and process substitution safely.

## Upstream pointers

- Repo: <https://github.com/mvdan/sh>
- Release notes: <https://github.com/mvdan/sh/releases>
- License: [BSD-3-Clause](https://github.com/mvdan/sh/blob/master/LICENSE) (`LICENSE`)
- Maintainer: [@mvdan](https://github.com/mvdan)
