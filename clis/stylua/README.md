# stylua

> **An opinionated Lua code formatter** — a single Rust binary that
> parses Lua 5.1 / 5.2 / 5.3 / 5.4 / Luau / Roblox-Luau into an AST
> and reprints it in a stable canonical shape, with a small set of
> knobs (column width, indent kind, quote style, call-parens policy).
> Pinned to **v2.4.1** (commit
> `631f02492a9477bc9cf9d8a1db2d471767fbe005`,
> [LICENSE.md](https://github.com/JohnnyMorganz/StyLua/blob/main/LICENSE.md),
> MPL-2.0).

Source: <https://github.com/JohnnyMorganz/StyLua>

## TL;DR

`stylua` is the `gofmt` / `rustfmt` / `prettier` of Lua: drop it on
your codebase, point it at a glob, and stop arguing about indent depth,
trailing commas in tables, single-vs-double quotes, and where to break
long function-call argument lists. It speaks every Lua flavour you
are likely to ship — vanilla 5.1 through 5.4, Luau (Roblox / Lune),
and the syntactic extensions (`continue`, type annotations, generalised
iteration) — selectable with a `--syntax` flag or a per-project
`stylua.toml`. It runs as a single ~6 MB static binary, has a
`--check` mode for CI, supports stdin → stdout for editor integration,
and is fast enough (the `full_moon` parser is hand-rolled, no PEG
backtracking surprises) to format a 50 KLOC repo in well under a
second.

## Install

```bash
# Homebrew (macOS / Linux)
brew install stylua

# Cargo
cargo install --locked stylua --features lua54
# (default features format Lua 5.2; pick lua51/lua52/lua53/lua54/luau as needed)

# npm (JS-tooling-friendly install)
npm install --save-dev @johnnymorganz/stylua-bin

# Linux package managers
# Arch (AUR): yay -S stylua-bin
# Nix:        nix-env -iA nixpkgs.stylua

# from a release tarball (any OS)
curl -LO "https://github.com/JohnnyMorganz/StyLua/releases/download/v2.4.1/stylua-macos-aarch64.zip"
unzip stylua-macos-aarch64.zip
sudo install stylua /usr/local/bin/

# verify
stylua --version    # stylua 2.4.1
```

## License

MPL-2.0 — see
[LICENSE.md](https://github.com/JohnnyMorganz/StyLua/blob/main/LICENSE.md).
File-level copyleft: modifications to existing `stylua` source files
must be released under MPL-2.0; *your* Lua source code is unaffected
(`stylua` only reads and rewrites it). Safe in commercial / closed
projects.

## One Concrete Example

```bash
# 1. format the whole tree in place
stylua .

# 2. check-only mode for CI (exits non-zero if anything would change)
stylua --check .

# 3. format from stdin (editor-on-save integration)
cat src/foo.lua | stylua -

# 4. format only what changed in the working tree (fast pre-commit)
git diff --name-only --diff-filter=ACMR | grep '\.lua$' | xargs stylua

# 5. project config (commit this to the repo)
cat > stylua.toml <<'TOML'
column_width                  = 100
line_endings                  = "Unix"
indent_type                   = "Spaces"
indent_width                  = 2
quote_style                   = "AutoPreferDouble"
call_parentheses              = "Always"
collapse_simple_statement     = "Never"
sort_requires                 = true
TOML

# 6. format Luau (Roblox / Lune) sources with type annotations
stylua --syntax Luau game/src/

# 7. exclude generated files
cat > .styluaignore <<'IGN'
build/
**/generated/*.lua
IGN

# 8. wire as a pre-commit hook (using pre-commit framework)
cat >> .pre-commit-config.yaml <<'YAML'
  - repo: https://github.com/JohnnyMorganz/StyLua
    rev: v2.4.1
    hooks:
      - id: stylua
YAML
```

## Niche It Fills

**Style-bikeshed eliminator for the entire Lua family in one binary.**
Lua's tooling ecosystem is fragmented across game-engine forks (Luau
for Roblox, LuaJIT for OpenResty, vanilla for Neovim plugins, Solar2D
for mobile games), and each historically had its own formatter or
none at all. `stylua` covers all of them through a `--syntax` flag
plus a single config file. It is the default formatter recommended by
the Roblox developer docs, the Neovim plugin community, and the Lune
runtime, which makes "what formatter do you use?" a non-question on
new Lua projects.

## Why use it

Three things `stylua` does that distinguish it from rolling your own
or using a generic linter's `--fix`:

1. **AST round-trip, not regex munging.** The `full_moon` parser
   produces a typed tree (statements, expressions, table fields, type
   annotations); the printer walks that tree and emits canonical
   tokens with consistent spacing, comma placement, and line breaks.
   You cannot accidentally break code by reformatting a multiline
   string or a comment-laden table the way regex-based reformatters
   do, and round-tripping `stylua | stylua` is a fixed point.
2. **Multi-syntax support is a single flag, not a fork.**
   `--syntax Lua54` accepts integer division and `goto`; `--syntax
   Luau` accepts type annotations, `continue`, and Roblox's generic
   iteration. The same binary, the same config file, the same
   editor plugin work across vanilla and Roblox codebases — important
   if you maintain a shared utility library that ships to both.
3. **Editor and CI story is solved.** First-party VS Code extension,
   Neovim integration via `null-ls` / `conform.nvim`, IntelliJ plugin,
   `pre-commit` hook, GitHub Action, plus `--check` for "fail PR if
   not formatted". Onboarding a new contributor reduces to "install
   `stylua` and turn on format-on-save".

For an LLM-CLI workflow that generates Lua (Neovim plugins, Roblox
gameplay scripts, OpenResty rate-limit modules), piping the model's
output through `stylua -` normalises whitespace and quote style
before the diff lands in `git`, so the human review focuses on
semantics rather than punctuation.

## Vs Already Cataloged

- **Vs [`dprint`](../dprint/):** orthogonal until they overlap.
  `dprint` is a multi-language formatter framework with plugins for
  TS / JS / JSON / Markdown / TOML / Dockerfile / Rust; its Lua
  support is via a `dprint-plugin-lua` that wraps `stylua` itself.
  Use `dprint` when you want one binary + one config to format a
  polyglot repo; use `stylua` directly when Lua is the dominant or
  only language and you want first-party releases without the
  plugin-version-pinning layer.
- **Vs [`ruff`](../ruff/) (Python) / [`biome`](../biome/) (JS/TS) /
  `gofmt` (Go) / `rustfmt` (Rust):** same role, different language.
  `stylua` is "the formatter for Lua" in the way `rustfmt` is "the
  formatter for Rust" — opinionated defaults, small knob set, AST
  round-trip, single binary.
- **Vs `lua-format` (LuaFormatter, not cataloged):** `lua-format` is
  the older C++ alternative; `stylua` ships more frequent releases,
  has wider Luau support, ships first-party editor integrations, and
  is the formatter the Roblox / Neovim / Lune communities have
  converged on. `lua-format` remains usable but is effectively legacy
  for new projects.
- **Vs `selene` (lints, not cataloged):** orthogonal.
  `selene` is a linter (catches `unused_variable`, shadowing,
  undefined globals); `stylua` is a formatter (rewrites whitespace
  and punctuation). Run both — they share a config conceptually
  (`selene.toml` + `stylua.toml`) and have non-overlapping verbs.

## Caveats

- **Few formatting knobs by design.** The maintainer pushes back on
  most "let me configure X" requests; if you hate the way `stylua`
  breaks long table constructors or wraps function arguments, the
  answer is usually "no, that is the canonical shape". This is the
  same posture as `gofmt` / `rustfmt`, but if your team's existing
  style guide prescribes something `stylua` does not support, you
  will be reformatting either the code or the style guide.
- **Comment placement can shift on first run.** The AST printer
  reattaches comments to the nearest meaningful token, which sometimes
  moves a comment up or down a line on the first format pass. Once
  the codebase is in canonical form, subsequent runs are fixed
  points; the noisy initial commit is a one-time cost.
- **Per-syntax feature flags at compile time.** The cargo-installed
  binary's syntax support is set by `--features lua51,lua54,luau,…`;
  the pre-built release tarballs and Homebrew bottles ship with all
  flavours enabled. If you build from source, double-check you
  enabled the syntax variant your project uses.
- **No semantic understanding.** `stylua` will not rename a variable,
  remove an unused import, or simplify `if x then return true else
  return false end` to `return x` — those are linter / refactor
  tasks. Pair with `selene` for catch-bugs work.
- **Roblox-specific syntax sometimes leads Luau spec.** The Roblox
  team ships syntax extensions ahead of upstream Luau spec stabilisation
  occasionally; `stylua` tracks fast but you may briefly see
  parse errors on the very newest Luau-only constructs until a
  release catches up. Workaround: pin `stylua` to the version that
  matches your Roblox studio release train.
