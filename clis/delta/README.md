# delta

> **A syntax-highlighting pager for git, diff, and grep output**
> — drop-in replacement for `less` as `core.pager` that
> re-renders `git diff` / `git log -p` / `git show` / `git blame`
> / `git stash show -p` (and bare `diff -u` / `rg --json` /
> `grep -n`) with side-by-side or unified layouts, true-color
> syntax highlighting via [`syntect`](https://github.com/trishume/syntect)
> using any [Sublime](https://www.sublimetext.com/) `.tmTheme`,
> per-hunk file headers, line numbers, navigable hunk-by-hunk
> jumps, and merge-conflict three-way visualisation. Pinned to
> **0.19.2** (commit
> `f85c46ba8b913aa3208af0f3573db90286e56e18`,
> [LICENSE](https://github.com/dandavison/delta/blob/main/LICENSE),
> MIT).

Source: <https://github.com/dandavison/delta>

## TL;DR

`delta` is the diff pager you wire into your `~/.gitconfig`
once and never think about again — every `git diff`, `git log
-p`, `git show`, `git stash show -p`, and `git blame` from
that point onward renders through it. The default theme adds
syntax highlighting (Rust, Python, TypeScript, Go, Markdown,
YAML, TOML, ~200 grammars from `syntect`), a per-file header
strip with the path + change summary, and right-aligned line
numbers in both the "before" and "after" columns. `delta
--side-by-side` switches from unified-diff layout to two
columns (left = before, right = after) which makes long
moved blocks readable; `delta --navigate` adds `n` / `N`
keybinds (forwarded to `less`) to jump between hunks; and
`delta --diff-so-fancy` emulates the popular `diff-so-fancy`
look for teams migrating from it. The same binary works on
the output of bare `diff -u`, `git diff` piped to a file,
`rg --json`, and merge conflict markers in a working-tree
file (it three-way-renders `<<<<<<<` / `=======` /
`>>>>>>>` blocks). Configuration lives in `~/.gitconfig`
under `[delta]` so the theme + features travel with the rest
of your git setup.

## Install

```bash
# Homebrew (macOS / Linux)
brew install git-delta

# Cargo
cargo install git-delta

# Linux package managers
# Arch:  pacman -S git-delta
# Debian / Ubuntu: apt install git-delta  (or download the .deb from releases)
# Fedora: dnf install git-delta

# wire it in (one-time)
git config --global core.pager delta
git config --global interactive.diffFilter "delta --color-only"
git config --global delta.navigate true            # n/N to move between hunks
git config --global delta.side-by-side true        # two-column layout
git config --global delta.line-numbers true        # right-aligned line nums
git config --global delta.syntax-theme "Monokai Extended"   # any .tmTheme name

# verify
delta --version    # delta 0.19.2
git diff           # syntax-highlighted, side-by-side, navigable
```

The wire-in step is what makes `delta` fire — without
`core.pager = delta` it just sits on disk. Once wired, every
`git diff` / `git log -p` / `git show` runs through it
automatically; no shell aliases to remember.

## License

MIT — see [LICENSE](https://github.com/dandavison/delta/blob/main/LICENSE).
Permissive; redistribute binaries freely; the `syntect`
dependency carries its own MIT license and bundled syntax
grammars carry their own (mostly MIT / Apache / BSD) — the
binary is fine to ship.

## Hot keybinds

`delta` itself has no keybinds — it forwards to `less` (or
whatever `$PAGER` resolves to). The keybinds that matter are
`less`'s, with two `delta`-specific additions when
`delta.navigate = true`:

- `n` — jump to next hunk / file (delta-specific marker
  injection makes this navigate file boundaries, not just
  pages)
- `N` — jump to previous hunk / file
- `j` / `k` / `↓` / `↑` — line down / up
- `Space` / `b` — page down / up
- `g` / `G` — top / bottom
- `/pattern` — search forward; `?pattern` — search backward;
  `n` / `N` continue search (clashes with delta's hunk jump
  unless you use `--navigate-regex` to scope it — usually
  fine, search wins inside a hunk)
- `q` — quit

For the inline `git --paginate=false diff | delta` invocation
(no pager wrapping) there are no keybinds — output streams to
stdout.

## Why use it

Three things `delta` does that the default `less`-piped diff
does not, that compound to "diff review on the terminal stops
hurting":

1. **Side-by-side layout for long moved blocks.** A diff that
   moves a 50-line function from one file to another is
   unreadable in the unified format (you see 50 lines of `-`
   followed by 50 lines of `+` and have to mentally pair
   them); side-by-side puts them on adjacent columns so the
   eye does the matching. `delta.side-by-side = true` makes
   every `git diff` use it; press `-` (less's resize key)
   to fit narrower terminals.
2. **Syntax highlighting that actually parses the language.**
   Through `syntect` and `.tmTheme` files (the same format
   Sublime / VS Code / Atom use), every `+` / `-` line gets
   real tokenization — Rust lifetime markers, Python f-string
   expressions, TypeScript JSX, Markdown headers all colored
   correctly. Reviewing an LLM-generated diff that touches
   five languages becomes possible without opening the files.
3. **Three-way merge conflict rendering.** When `delta`
   encounters `<<<<<<< HEAD` / `||||||| common` / `=======` /
   `>>>>>>> branch` markers in piped diff output (e.g.
   `git diff` during a conflicted merge), it renders three
   columns: ours / base / theirs, with the diff between each
   pair highlighted. The "stare at conflict markers in the
   editor and try to remember which side was which" failure
   mode goes away.

For an LLM-CLI workflow that produces large multi-file diffs
that you scroll through to spot regressions before staging,
`delta` makes the scroll-through significantly faster — the
side-by-side layout + syntax highlighting + per-hunk navigate
keys are the difference between "I will skim this and probably
miss something" and "I will actually read this".

## Vs Already Cataloged

- **Vs [`lazygit`](../lazygit/) / [`gitui`](../gitui/):** different
  layer. Those are *git clients* (status / stage / commit /
  rebase UIs); `delta` is a *diff renderer* that those clients
  shell out to when `core.pager = delta`. They compose: set
  `core.pager = delta` and `lazygit`'s "view commit diff"
  modal automatically renders through it.
- **Vs [`bat`](../bat/):** `bat` is a `cat` replacement with
  syntax highlighting (it does not understand diff format —
  feed it a `.diff` file and it highlights the diff *syntax*,
  not the underlying source language on each `+` / `-` line).
  `delta` parses diff structure and applies the underlying
  language's grammar to the changed lines. Use `bat` for
  reading source files; use `delta` for reading diffs.
- **Vs [`git-diff` builtin --color-words / --word-diff`:**
  those are git's own intra-line diff modes — useful for prose,
  noisy for code. `delta` does intra-line via `delta.features
  = side-by-side line-numbers decorations` with character-level
  diff inside each changed line, which is sharper than
  `--color-words` on code without the noise.
- **Vs `diff-so-fancy`:** the predecessor that inspired
  `delta`'s default look. `diff-so-fancy` is Perl-only and
  does no syntax highlighting; `delta --features=diff-so-fancy`
  emulates its visual style with the syntax-highlighting +
  side-by-side bonuses on top, so migrating is a config
  change.

## Caveats

- **`core.pager = delta` runs delta on every paginated git
  command** — including `git log` (no `-p`), where it has
  nothing diff-shaped to render and just forwards to `less`.
  This is correct (no perf hit) but if you scripted around the
  exact `less` behavior of `git log` (e.g. piping to `head`)
  you may see different terminal sequences. Set
  `delta.paging = never` for those scripts or pass
  `--paginate=false`.
- **`syntect` highlighting is heuristic** — files without a
  recognised extension fall back to plain text; files with a
  shebang in a hunk that doesn't include the first line lose
  language detection for that hunk. Acceptable trade-off; can
  be forced with `delta --syntax-theme=... --default-language=Rust`
  in a script.
- **`delta` is not a `git diff` replacement** — it is a
  *renderer* that consumes `git`'s diff output. If you set
  `delta.features` that conflict with `git`'s diff settings
  (e.g. `delta` expects `--color=always` but a script forces
  `--color=never`), output looks broken. The recommended
  invocation pattern is to leave `git diff` defaults alone and
  let the `core.pager = delta` wiring handle it.
- **Performance on huge diffs (10k+ lines)** is fine for
  interactive review but slower than raw `less` because every
  changed line is tokenized and rendered. For a `git log -p`
  over thousands of commits, prefer `git log --stat` or scope
  with `git log -p -- <path>`.
- **`delta --side-by-side` needs ~160 columns** to be
  comfortable — narrower terminals get cramped two-column
  layout where each side wraps. Use `delta.side-by-side =
  false` on a laptop screen + `true` on a wide external
  monitor by setting it per-machine in
  `~/.config/git/config-local` and including it from
  `~/.gitconfig`.
