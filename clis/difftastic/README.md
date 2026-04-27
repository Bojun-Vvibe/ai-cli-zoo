# difftastic

> **A structural diff tool that understands syntax.** Pinned to
> **0.68.0**, MIT
> ([LICENSE](https://github.com/Wilfred/difftastic/blob/master/LICENSE)).

- **Repo:** https://github.com/Wilfred/difftastic
- **Latest version:** 0.68.0
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `git-tools` / `dev-experience`
- **Language:** Rust

## What it does

`difftastic` (binary name `difft`) is a tree-sitter–powered diff tool
that compares files based on their *syntax tree* rather than their
line layout. Instead of a line-oriented hunk that highlights every
shifted whitespace token when you indent a block, `difft` parses both
sides with the appropriate tree-sitter grammar (50+ languages
including Rust, Go, TypeScript, Python, Ruby, Java, C/C++, JSON,
YAML, TOML, HTML, CSS, Markdown), aligns matching subtrees, and
reports only the nodes that genuinely changed. The result reads as
"the function `foo` had an `if` branch added" instead of
"lines 42–58 are red, lines 42–61 are green." Drops in to git via
`git config --global diff.external difft` or as a per-command pager
(`GIT_EXTERNAL_DIFF=difft git diff`); also runs standalone on two
files. Side-by-side and inline output modes both colour-aware.

## Why included

Closes the "what actually changed" gap that line-based diff has
suffered with for 50 years. Pairs naturally with [`delta`](../delta/)
(which is still better for sheer diff *volume* and `git log -p`
review) — `difft` for surgical "did the refactor preserve
semantics?" inspection, `delta` for everyday browsing. One of the
clearest "this is a category-creating tool" Rust CLIs of the last
five years.
