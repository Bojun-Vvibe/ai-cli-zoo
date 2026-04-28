# serpl

- **Repo:** https://github.com/yassinebridi/serpl
- **Version:** 0.3.4 (latest stable, 2025)
- **License:** MIT ([LICENSE](https://github.com/yassinebridi/serpl/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install serpl` · `cargo install serpl` · prebuilt binaries on the GitHub release page · binary name is `serpl` (Search and Replace TUI)

## What it does

`serpl` is a terminal UI for **interactive search-and-replace across
a directory tree**. It opens on the current directory, you type a
search pattern (literal, regex, or whole-word), then a replacement
pattern, and it renders a live, scrollable list of every matched
file with the affected lines, before / after diff colouring, and
per-match toggle checkboxes. You walk the list, deselect any match
you don't want rewritten, and hit `Enter` to apply the rest.
Backends are swappable: by default it drives `ripgrep` for the
search half, but it can also use `ast-grep` for structural matches
when you need to rewrite by AST node instead of by line. Filters
respect `.gitignore` and let you scope by glob, file type, or
fuzzy-picked subdirectory.

## When to pick it / when not to

Pick `serpl` when you need to do a rename / refactor that `sed -i`
*almost* handles but where you need to **eyeball each match** —
e.g. renaming a function whose name appears in a few unrelated
contexts you want to leave alone, fixing a typo across a docs tree
where some occurrences are in code samples that must not change,
or applying a regex rewrite where the replacement is correct in
80% of cases and you want to skip the rest. The TUI gives you the
"check every diff before I touch the disk" loop without piping
through `fzf` and `xargs sed`.

Skip it when the rewrite is **uniform and unambiguous** — `sd`,
`sed -i`, or `rg --replace | sponge` are faster and scriptable.
Skip it when the rewrite is **semantic / structural** at scale
(rename a symbol respecting scope) — language servers via
`code --rename`, `rust-analyzer`, or
[`ast-grep`](../ast-grep/) used directly are safer than line-based
replace. Skip it for **binary** files, very large monorepos where
the up-front match list would be unwieldy, or any rewrite that
must be reproducible in CI (a TUI is not a build step).

## Why it matters in an AI-native workflow

Agents are good at proposing a regex or `ast-grep` pattern but
historically bad at *applying* it surgically — a one-character
mistake in the replacement and you've corrupted dozens of files,
which then have to be reverted from `git`. `serpl` is the human-
or agent-in-the-loop checkpoint: the agent emits the pattern + the
proposed replacement, the human (or a reviewing agent) opens the
TUI, scans the per-match diffs, deselects the bad ones, and
commits. The on-disk write happens once, atomically per file, only
on confirm. Combined with `git` it gives you a tight "propose →
review → apply → diff → amend" loop that beats letting the agent
shell out to `sed -i` directly.

## Example invocations

```bash
# Open in the current directory with the ripgrep backend (default)
serpl

# Open scoped to a subdirectory, with a starting search pattern
serpl --project-root ./src --search-pattern 'old_function_name'

# Use ast-grep as the search backend (structural, respects syntax)
serpl --search-backend ast-grep

# Typical TUI flow:
#   /          focus the search field; type the pattern
#   r          toggle regex / literal / word-boundary mode
#   Tab        move to the replacement field; type the replacement
#   j / k      walk the match list; Space toggles a single match
#   d          deselect every match in the highlighted file
#   p          live preview of the diff for the highlighted match
#   Enter      apply selected matches to disk (one write per file)
#   u          undo the last apply (reverts in-process)
#   ?          help overlay
#   q          quit without applying

# Pair with git so every apply is followed by a reviewable diff:
git status && git diff
git add -p && git commit -m 'refactor: rename old_function_name -> new_name'
```

## Alternatives in this catalog

- [`sd`](../sd/) — non-interactive `sed` replacement with an
  intuitive pattern syntax; pick `sd` when the rewrite is uniform
  and you don't need to eyeball matches.
- [`ripgrep`](../ripgrep/) — `rg --passthru --replace` does
  non-interactive replace if you pipe through `sponge`; the
  scriptable cousin of `serpl`.
- [`ast-grep`](../ast-grep/) — structural search/replace by AST
  pattern; `serpl` can wrap it as a backend, or you can drive
  `ast-grep` directly with `--rewrite`.
- [`fastmod`](../sd/) — Meta's interactive sed-style tool with a
  similar y/n-per-hunk loop on the CLI; `serpl` is the TUI
  equivalent with multi-file overview.
- [`broot`](../broot/) — different niche (filesystem navigation)
  but pairs well as the file-picker that scopes a `serpl` run.
