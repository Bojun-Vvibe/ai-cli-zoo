# tre-command

> **A `tree(1)` rewrite that is gitignore-aware
> by default *and* writes shell aliases (`$e1`,
> `$e2`, …) for each entry it prints, so you can
> open a deep file by typing two characters
> instead of re-typing the whole path.** Pinned
> to **v0.4.0**
> ([LICENSE](https://github.com/dduan/tre/blob/main/LICENSE),
> MIT).

Source: <https://github.com/dduan/tre>

## TL;DR

The classic `tree` does one job (list a
directory hierarchy) and is unaware of
`.gitignore`, so it floods you with
`node_modules/`, `target/`, `.venv/` etc.
[`erdtree`](../erdtree/) and `eza --tree`
fix that, but `tre`'s superpower is the
`-e/--editor` flag: every printed entry gets
an indexed shell alias, so right after `tre
-e`, you can run `e7` to open the seventh
file in `$EDITOR`. It turns `tree` from a
print-only display tool into a navigable
directory picker — much like `fzf`-into-
editor, but with persistent visual context.

## Install

```bash
# cargo
cargo install tre-command

# Homebrew
brew install tre-command

# Pre-built binary
curl -LO https://github.com/dduan/tre/releases/download/v0.4.0/tre-v0.4.0-x86_64-apple-darwin.tar.gz
tar xzf tre-v0.4.0-x86_64-apple-darwin.tar.gz
sudo install tre /usr/local/bin/

# IMPORTANT: source the alias-emitter shim once per shell
# (auto-writes $e1..$eN aliases on every `tre -e` run)
echo 'tre() { command tre "$@" -e $TRE_EDITOR && source /tmp/tre_aliases_$$ 2>/dev/null }' >> ~/.zshrc

tre --version    # tre 0.4.0
```

## Use it for

```bash
# Plain tree, gitignore-respecting
tre

# Limit depth
tre -l 2

# Print + create $e1, $e2, ... aliases (the killer feature)
tre -e
# → outputs the tree with indices like:
#     1 src/
#     2   main.rs
#     3   lib.rs
#     4 Cargo.toml
# Then:
e2          # opens src/main.rs in $EDITOR
e4          # opens Cargo.toml

# Include hidden + ignored files
tre -a

# Exclude a pattern (in addition to .gitignore)
tre -E 'target|*.lock'

# JSON output (machine-readable)
tre -j | jq '.[] | select(.type=="file")'
```

## Why include it in a CLI catalog

1. **The aliasing trick is genuinely
   novel.** Most "tree-then-pick" workflows
   pipe through `fzf`, which collapses the
   visual tree into a flat list. `tre -e`
   keeps the tree on screen *and* gives you
   a one-keystroke jumper — best of both.
2. **Gitignore-aware by default.** Same
   sensible default as
   [`erdtree`](../erdtree/) /
   [`broot`](../broot/) — no more 50,000-
   line `tree` outputs in a Node project.
3. **Tiny, single-file Rust binary.** No
   runtime, no plugin system, no config —
   one binary on every major platform.

## Vs Already Cataloged

- **Vs [`erdtree`](../erdtree/):** `erdtree`
  is the more featureful "modern tree" —
  rich sorting, sizes, du-style aggregation.
  `tre` is smaller and adds the editor-
  alias trick that `erdtree` lacks.
- **Vs [`broot`](../broot/):** `broot` is a
  full-screen TUI navigator (fuzzy match,
  preview, verbs). `tre` is a one-shot
  printer + alias emitter that composes with
  your normal shell. Use `broot` for
  exploration; use `tre -e` when you already
  know roughly what you want and just need a
  fast way to open it.
- **Vs `tree` (BSD/GNU):** `tre` is
  gitignore-aware out of the box, has the
  alias trick, and emits JSON. It does not
  cover every classic `tree` flag — for
  perfect drop-in compat, keep `tree`
  installed alongside.

## Caveats

- The alias trick requires a tiny shell
  function wrapper (the snippet in
  *Install*); the binary alone only prints
  numbered output to stderr.
- `$TRE_EDITOR` must be set (defaults to
  `$EDITOR`) for `e1`, `e2`, … to work.
- v0.4.0 is the latest tagged release
  (June 2022); the project is feature-
  complete rather than abandoned, but
  expect no churn.
- Aliases are scoped to the shell session
  in which `tre -e` ran; spawning a new
  shell loses them (rerun `tre -e`).
