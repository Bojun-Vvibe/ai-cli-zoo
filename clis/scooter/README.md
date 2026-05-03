# scooter

> **Interactive find-and-replace, in a TUI** — the missing
> middle ground between `sed -i` (no preview, easy to wreck a
> repo) and an editor's "replace in files" panel (locked to one
> editor, slow on huge trees): a single Rust binary that walks
> the current directory, runs your search across every file in
> parallel, lets you toggle each individual match on or off in a
> full-screen list with diff context, and only writes to disk
> after you confirm — with a real undo on top. Pinned to
> **v0.9.1** (commit
> `6e47ed05feeef605d076a9e1ac1aaac159df3886`,
> [LICENSE](https://github.com/thomasschafer/scooter/blob/main/LICENSE), MIT).

Source: <https://github.com/thomasschafer/scooter>

## TL;DR

Run `scooter` in any directory. A two-pane TUI opens: top is the
search box (regex by default, with case-sensitivity, fixed-string,
and whole-word toggles), bottom is the replacement box, plus an
optional file-glob filter (`**/*.go`, `!**/vendor/**`). Hit
`Enter` and `scooter` walks the tree (respecting `.gitignore` via
`ignore`/`ripgrep`'s walker), shows every match grouped by file
with three lines of context and a colored before/after diff, and
puts a checkbox on each one. You space-toggle individual matches,
`a` to select all in a file, `A` to select all globally, then
press `Enter` again to write. The writes happen atomically per
file (temp-file + rename), and `scooter` keeps a per-run undo
journal so `u` rolls everything back. Capture groups (`$1`, `$2`,
`${name}`) work in the replacement; so do escapes (`\n`, `\t`).
You can also drive it non-interactively with
`scooter --search foo --replace bar --no-tui` for scripts that
want the same walker but no UI.

## Install

```bash
# Homebrew
brew install scooter

# Cargo
cargo install scooter --locked

# Pre-built binaries (Linux x86_64/aarch64, macOS arm64/x86_64, Windows)
gh release download v0.9.1 -R thomasschafer/scooter
# or grab from
# https://github.com/thomasschafer/scooter/releases/tag/v0.9.1

# Nix
nix run nixpkgs#scooter

# verify
scooter --version    # scooter 0.9.1
```

Sample first run:

```bash
cd ~/code/my-project
scooter
# search box:        getUserById\((\w+)\)
# replace box:       fetchUser($1)
# include glob:      **/*.{ts,tsx}
# exclude glob:      !**/node_modules/**  !**/dist/**
# Enter -> review every match, space-toggle the wrong ones, Enter to write
# u -> undo entire run
```

## Why it's worth a slot in the zoo

Bulk find-and-replace is one of those operations everyone does
weekly and nobody has a great workflow for. `sed -i 's/old/new/g'
$(rg -l old)` is the classic one-liner — fast, scriptable, and
also the fastest way to silently corrupt a string inside a JSON
fixture, a comment, or a Markdown code fence that you did not mean
to touch. Editor "replace in files" works but locks you into one
editor and stalls on monorepos with a hundred thousand files.
`scooter` is the rare tool that combines the *speed* of `ripgrep`'s
walker (it uses the same `ignore` crate, honors `.gitignore` and
`.ignore`) with the *safety* of an editor's preview-then-confirm
flow, and makes per-match opt-in cheap. The fact that it is a
single 5 MB binary with no language runtime, no config, and a real
undo makes it the tool I now reach for before `sed` for anything
non-trivial.

## Where it sits

- vs `sed -i` / `perl -pi -e`: `scooter` is a pre-flight UI on top
  of the same idea. Use `sed` in CI and one-shot pipelines, use
  `scooter` when a human is in the loop.
- vs [`fastmod`](https://github.com/facebookincubator/fastmod):
  `fastmod` is closest in spirit — a Rust tool that prompts
  per-match in the *same* terminal stream. `scooter` improves on
  it with a full-screen TUI that lets you scroll back, multi-select,
  re-edit the regex without restarting, and undo.
- vs [`amber`](https://github.com/dalance/amber) / `sad`: those are
  also "preview, then write" Rust tools, but their preview is a
  scrolling diff dump, not an interactive checklist. `scooter`'s
  per-match toggle is the differentiator.
- vs `git grep -l ... | xargs sed -i`: the bash classic, fine
  for trivial cases. Falls apart the moment one file should keep
  the old string (e.g. a CHANGELOG entry, a snapshot test).
- vs editor refactor (VS Code "Replace in Files", IntelliJ
  "Refactor → Rename"): editor flows are tied to a language server
  for true rename. `scooter` is *textual* — use it for strings,
  config keys, doc references, and anywhere the LSP cannot help.

## Footguns

- It is text-level, not AST-level. Renaming an identifier with
  `scooter` will also touch the same string inside a comment,
  test fixture, or unrelated import — that is *why* the per-match
  toggle exists. Use `tsserver`/`gopls`/`rust-analyzer` rename for
  true semantic refactors.
- The `--no-tui` mode skips the per-match prompt entirely and
  writes everything that matches; treat it like `sed` and dry-run
  it first (`--dry-run`).
- Undo is per-run and lives in memory; quit `scooter` after a
  write and the journal is gone. For real safety net, commit the
  pre-state to git first.
- The walker honors `.gitignore` by default. If you need to
  rewrite vendored code or generated files, pass
  `--no-ignore` / `--hidden` explicitly.
- Replacement strings with `$` are interpreted as backrefs; to
  insert a literal `$`, escape as `$$`. The TUI does not warn you
  if `$1` is referenced without a corresponding capture group —
  the replacement just becomes empty.
- On binary files (matched by content sniffing) `scooter` skips
  rather than corrupting; if your "text" file has a stray null
  byte, it gets skipped silently — pass `--binary` to force.
