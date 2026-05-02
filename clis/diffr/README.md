# diffr

> **Word-level diff highlighter that wraps any line-level diff.**
> A small Rust filter that reads unified-diff output on stdin and
> re-emits it with intra-line additions/deletions colored, so you
> see *which characters* changed inside a `-`/`+` line pair, not
> just that the whole line changed.
> Pinned to **v0.1.5**
> ([LICENSE](https://github.com/mookid/diffr/blob/master/LICENSE),
> MIT).

Source: <https://github.com/mookid/diffr>

## TL;DR

`diffr` is a pipe-friendly post-processor for `git diff`, `diff -u`,
`hg diff`, `svn diff`, or any other unified-diff producer. It runs
a Patience-style token diff inside each `-` / `+` hunk pair and
colors the differing tokens. The line-level structure is left
untouched, so it composes with `less -R`, `delta`, pagers, code
review tools — anything that already understands ANSI-colored
unified diffs.

## Install

```bash
# Cargo (any platform with a Rust toolchain)
cargo install diffr

# Homebrew
brew install diffr

# Pre-built binaries
# https://github.com/mookid/diffr/releases/latest

# verify
diffr --version    # diffr 0.1.5
```

## License

MIT — see
[LICENSE](https://github.com/mookid/diffr/blob/master/LICENSE).
Permissive: bundle, fork, or ship inside a closed product without
extra obligations beyond preserving the notice.

## One Concrete Example

```bash
# Wire it into git as the default pager for `git diff` / `git show`
git config --global core.pager 'diffr | less -R'

# Or one-shot, without changing config
git diff | diffr | less -R

# Plain unified diff against a backup
diff -u config.yaml.bak config.yaml | diffr

# Inside a code review: a one-character typo fix that would
# otherwise look like the whole line changed
#   - assert response.status_cdoe == 200
#   + assert response.status_code == 200
# diffr highlights only `cdoe` -> `code`, the rest is dim.
```

## Niche It Fills

**The "this `-`/`+` pair differs by three characters but the
default red/green paints the entire line" gap.** For long lines
(SQL, JSON, prose, generated code), line-level coloring buries the
actual edit. `diffr` keeps the unified-diff format your tools
already understand and just adds the missing intra-line layer.

## Why use it

1. **Pipe-in, pipe-out.** No config, no daemon, no repo awareness.
   If something produces a unified diff, `diffr` will color it.
2. **Stays out of the format.** Unlike full diff replacements,
   `diffr` preserves hunks, headers, and line counts, so review
   tools and `patch(1)` keep working downstream.
3. **One small Rust binary.** No runtime, no plugins, drops into
   `~/.local/bin` and never breaks on upgrade.

## Vs Already Cataloged

- **Vs [`bat`](../bat/):** `bat` is a syntax-highlighting `cat`,
  not a diff tool. Complementary — `bat` colors files at rest,
  `diffr` colors changes between files.
- **Vs `delta` (not yet cataloged):** `delta` is a full diff
  reskin with side-by-side mode, line numbers, and theming.
  `diffr` is a much smaller, do-one-thing filter you stack on top
  of whatever pager you already use.

## Caveats

- **Token diff is heuristic.** Very long lines or near-total
  rewrites can still highlight most of the line — that's correct
  behavior, but visually similar to plain line-level diff.
- **Needs a TTY-aware downstream.** Pipe through `less -R` (not
  plain `less`) or write to a terminal; otherwise the ANSI codes
  end up as literal escape sequences.
- **Unified diff only.** Context diff (`diff -c`) and side-by-side
  (`diff -y`) input are not supported — convert to `-u` first.
