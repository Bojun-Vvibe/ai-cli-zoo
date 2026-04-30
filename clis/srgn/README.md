# srgn

> **A grep-like *code surgeon* — search AND mutate source code by
> syntactic structure, not by line regex.** Combines `tree-sitter`
> language grammars with `ripgrep`-style search and `sed`-style
> in-place rewriting; selects only the syntactic regions that matter
> (function bodies, comments, string literals, type signatures) and
> applies the action there. Pinned to **srgn-v0.14.2**
> ([LICENSE](https://github.com/alexpovel/srgn/blob/main/LICENSE),
> MIT).

Source: <https://github.com/alexpovel/srgn>

## TL;DR

`grep` / `sed` see text; `srgn` sees a parse tree. Pass
`--python 'function-names@^test_'` and you are scoped to function
*identifiers* matching the regex inside Python files only — not
matches in comments, strings, or anywhere else those characters
happen to occur. Stack scopes to narrow further:
`--python comments --python urls` selects URLs found inside
comments. Once scoped, you can search (`grep` mode, default),
replace (`'pattern' 'replacement'` positional args), or run any of
the built-in transformers — `--upper`, `--lower`, `--titlecase`,
`--squeeze` (collapse runs of whitespace), `--delete`, `--normalize`
(NFD), `--german` (umlaut handling). Languages with first-class
scopes at v0.14: Python, Rust, Go, TypeScript, Java, C, C++, C#,
HCL, Bash, PHP, Ruby, Elixir, Lua, Solidity. Fundamentally the tool
"replace `Foo` with `Bar` only inside type signatures of Rust
functions, never in string literals or doc comments" was previously
a multi-hour ad-hoc script and is now a one-liner.

## Install

```bash
# Homebrew (macOS / Linux)
brew install srgn

# Cargo (any platform)
cargo install --locked srgn

# Arch Linux
pacman -S srgn

# Pre-built binary
curl -L https://github.com/alexpovel/srgn/releases/download/srgn-v0.14.2/srgn-x86_64-unknown-linux-gnu.tar.gz | tar xz
```

## Example

```bash
# Find all uses of a deprecated function name across Python *call sites*
# (not in strings, not in comments)
srgn --python 'function-calls' 'old_helper'

# Rename a Rust struct field, but only in field accesses (not in any
# unrelated identifier of the same spelling)
srgn --rust 'identifiers@^old_field$' 'new_field' --files 'src/**/*.rs'

# Strip TODO comments from a Go codebase, recursively, in-place
srgn --go comments 'TODO.*' '' --files '**/*.go'
```

## When to use

- A refactor would be trivial except `sed` keeps mangling string
  literals or matching identifiers inside doc comments — srgn lets
  you exclude those scopes by construction.
- You want a "rename in language-aware scope" without standing up an
  LSP / language server (works on read-only repos, no build needed).
- Codebase-wide audits like "every URL hard-coded inside a Python
  comment" or "every function whose body contains both `unsafe` and
  `transmute`".

## When NOT to use

- You need *semantic* rename (rename a symbol *and* its definition
  *and* every reference, respecting shadowing / imports). That is
  an LSP rename — use `rust-analyzer`, `gopls`, etc. via your
  editor.
- The language is not in srgn's tree-sitter grammar list yet.
- You only need a plain regex over plain text — `ripgrep` + `sd`
  is faster to type and does not need a grammar parse.
