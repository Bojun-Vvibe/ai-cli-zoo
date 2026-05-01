# mergiraf

## What it does
A **syntax-aware git merge driver** that resolves three-way merges by parsing
all three revisions (base, ours, theirs) with **tree-sitter grammars** and
merging at the AST level instead of the line level. When two branches add
imports to a Python file, reorder methods inside a class, both edit different
TOML keys, or independently insert items into a JSON array, line-based `git
merge` reports a conflict; mergiraf recognises that those edits **commute** at
the syntax level and merges them cleanly, leaving conflict markers only for
edits that actually overlap semantically. Wire it up with `mergiraf languages
--git-attributes` (writes a `.gitattributes` block declaring the merge driver
per supported language) and `git config merge.mergiraf.driver "mergiraf merge
%O %A %B -s %S -x %X -y %Y -p %P -l %L"`. Also ships `mergiraf solve <file>`
for retroactively solving a file that already has conflict markers, and
`mergiraf review <commit>` to audit a past merge commit. Supported languages
include Rust, Python, TypeScript / TSX, JavaScript / JSX, Go, Java, C, C++,
C#, Ruby, Kotlin, Scala, Swift, Haskell, Elixir, JSON, YAML, TOML, XML, HTML,
CSS, Markdown, Dart, Lua, Verilog / SystemVerilog, GNU Make, CMake, Starlark,
and `pyproject.toml`.

## Why it's interesting
Different shape from `git merge` (line-based, fast, no AST awareness — the
default), `mergetool` plumbings like `meld` / `kdiff3` / `vimdiff` (interactive
GUI / TUI conflict resolvers — they show you the conflict, they do not avoid
it), `git-imerge` (incremental merge that bisects to find the smallest
conflicting commits — orthogonal: it shrinks the conflict surface, not the
conflict itself), and IntelliJ's semantic merge (closed-source, IDE-locked,
JVM-only). mergiraf is the *open-source AST-merge-as-a-git-driver* option: it
is a single Rust binary you point `git` at via `.gitattributes`, it falls back
to `git merge-file` on parser errors so it never makes things worse, and it
ships a `mergiraf review` command so you can sanity-check what the AST merge
actually did. Choose it when your team's PR queue is being throttled by
trivial-but-frequent import / class-method / TOML-key conflicts on long-lived
branches; do **not** choose it as a substitute for actually rebasing — it
reduces the noise floor, it does not fix the underlying long-branch problem.
Project is Codeberg-hosted and FSF-listed.

## Niche category
Syntax-aware three-way git merge driver — tree-sitter AST merging with
line-merge fallback, runs as the configured `merge.driver` for matching
files.

## Repo
https://codeberg.org/mergiraf/mergiraf

## Version pinned
`v0.16.3` (latest stable release, 2026-01-26)

## License
- SPDX: `GPL-3.0-or-later`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linuxbrew)
brew install mergiraf

# Cargo
cargo install mergiraf --locked
# or pre-built binary via cargo-binstall
cargo binstall mergiraf

# Debian package (since 0.16.x: `cargo deb` from source)
# Arch
sudo pacman -S mergiraf

# Pre-built tarballs:
# https://codeberg.org/mergiraf/mergiraf/releases
```

## Usage examples
```sh
# One-time, per-clone setup: register mergiraf as a git merge driver
git config --global merge.mergiraf.name "mergiraf"
git config --global merge.mergiraf.driver \
  "mergiraf merge %O %A %B -s %S -x %X -y %Y -p %P -l %L"

# Per-repo: declare which file types should use it (writes .gitattributes)
mergiraf languages --git-attributes >> .gitattributes
git add .gitattributes && git commit -m "chore: enable mergiraf"

# From now on, normal `git merge` / `git rebase` / `git cherry-pick` use
# mergiraf for the declared file types and fall back to git's own driver
# on parser error
git merge feature/refactor-imports

# Retroactively solve a file that already has <<<<<<< markers in it
mergiraf solve src/foo.py

# Audit what the AST merge actually produced vs what a line merge would have:
mergiraf review HEAD              # last merge commit
mergiraf review abc1234           # any merge commit by SHA

# See which languages mergiraf currently knows how to parse and merge
mergiraf languages

# One-off merge of three explicit files (useful in CI / scripts)
mergiraf merge base.json ours.json theirs.json -p path/in/repo.json
```
