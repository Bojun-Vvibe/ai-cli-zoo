# bfs

- **Repo:** https://github.com/tavianator/bfs
- **Version:** 4.1.1 (released 2026-04-29)
- **License:** 0BSD ([LICENSE](https://github.com/tavianator/bfs/blob/main/LICENSE))
- **Language:** C
- **Install:** `brew install bfs` · `apt install bfs` · `pacman -S bfs` · `port install bfs` · build from source with `./configure && make` · binary name is `bfs` (Breadth-First Search)

## What it does

`bfs` is a drop-in replacement for the POSIX `find(1)` utility that
walks the filesystem **breadth-first** instead of depth-first. The
command-line surface is a strict superset of `find`: every primary you
already know (`-name`, `-type`, `-mtime`, `-perm`, `-exec`, `-prune`,
`-newer`, `-size`, `-delete`, …) works exactly the same way and
returns exactly the same set of paths, just in a different (shallower-
first) order. On top of that, `bfs` adds quality-of-life features that
`find` has resisted for thirty years: colored output that follows
`LS_COLORS`, an optional argument-order *fixer* (`-D opt` shows where
your filter could be cheaper), `-files0-from` for safe NUL-separated
path lists, `-exclude` as a real top-level operator instead of the
`-prune -o ... -print` dance, regex flavours selectable per
invocation, and predicates `find` never grew (`-hidden`, `-nohidden`,
`-unique`, `-xattr`, `-status`).

## When to pick it / when not to

Pick `bfs` when you want **shallowest-match-first ordering** — looking
for the nearest `package.json` from `cwd`, or the nearest `.git`
directory above a deeply nested file, or the first `node_modules` to
delete in a monorepo. Depth-first `find` will descend into a 12-level
deep `vendor/` tree before it ever notices the `package.json` in the
current directory; `bfs` returns the shallow hit first, so piping
through `head -1` is a fast "find the closest" query. Also pick it
when you want `find`'s exact predicate language but with colored
output and the `-exclude` operator written the way humans expect.

Skip it when you want a faster **glob-style** search over a project
tree — [`fd`](../fd/) is the better reach there (smaller surface,
parallel walker, `.gitignore` aware by default). Skip it when you
need GNU-`find`-only extensions that `bfs` deliberately omits, or
when you are scripting against the historical depth-first ordering
(use `-S dfs` to restore it explicitly so future readers see the
intent).

## Why it belongs in the zoo

The zoo already has `findutils` (the POSIX baseline) and `fd` (the
modern Rust glob-walker). `bfs` is orthogonal to both: it keeps
`find`'s exact CLI grammar (so existing `find ... -exec ... \;`
scripts work unchanged) but flips the **traversal order** to
breadth-first, which is a different algorithmic property neither
neighbour offers. The 0BSD license is also notable — strictly more
permissive than MIT, with no attribution requirement, which makes it
trivial to vendor into proprietary build pipelines.

## Example invocations

```bash
# Find the nearest package.json walking up from cwd's subtree
bfs . -name package.json | head -1

# Same predicates as find, but colored and with -exclude as a real op
bfs ~/src -exclude -name node_modules -name '*.test.ts'

# Show how bfs would re-order the predicates for cheapness
bfs -D opt . -name '*.log' -mtime +30 -size +10M

# NUL-separated list of inputs (safe with newlines in filenames)
printf '%s\0' file1 'weird name\nwith newline' | \
  bfs -files0-from - -type f -printf '%p\n'

# Force depth-first if a script depends on historical find order
bfs -S dfs . -type f -name '*.bak' -delete
```
