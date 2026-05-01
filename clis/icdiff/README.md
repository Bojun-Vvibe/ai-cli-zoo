# icdiff

> **Side-by-side colored diff for the terminal** — a single
> Python script that takes two files (or stdin) and renders a
> two-column diff with intra-line highlighting (the changed
> *characters* inside a changed line are bolded), full Unicode
> width handling for CJK / emoji, and an output that fits the
> actual terminal width (auto-detected, override with
> `--cols=N`). Ships with a `git-icdiff` wrapper that drops in
> as a `git difftool` so `git icdiff HEAD~3 -- src/` becomes the
> default diff view. Pinned to **release-2.0.10** (commit
> `e8dd4a3041baeb66a11589e308362608c88cdd47`,
> [LICENSE](https://github.com/jeffkaufman/icdiff/blob/release-2.0.10/LICENSE),
> PSF License v2 — BSD-compatible permissive).

Source: <https://github.com/jeffkaufman/icdiff>

## TL;DR

`icdiff` is the answer to "GNU `diff` is unreadable when more
than two adjacent lines change, and `diff -y` doesn't show
*which characters* in the line moved." It splits the screen,
shows old on the left and new on the right with line numbers,
and within each changed line highlights only the diverged
characters in red / green so a one-character typo doesn't look
like a whole-line rewrite. Pure Python, no compiled deps,
single-script — drops onto any Python-3 box (`pip install
icdiff` or just `curl` the script).

## Install

```bash
# pipx (recommended — isolates the script in its own venv)
pipx install icdiff

# Homebrew (macOS / Linux)
brew install icdiff

# pip
pip install --user icdiff

# Arch:    pacman -S icdiff
# Debian:  apt install icdiff
# Nix:     nix-env -iA nixpkgs.icdiff

# verify
icdiff --version    # icdiff 2.0.10
```

`git-icdiff` (the git wrapper) ships in the same package and
ends up next to `icdiff` on `$PATH`.

## License

PSF License v2 — see
[LICENSE](https://github.com/jeffkaufman/icdiff/blob/release-2.0.10/LICENSE).
The PSF license is BSD-style permissive; redistribution requires
keeping the notice but otherwise places no obligation on
downstream code (the author chose PSF because the original
algorithm came from CPython's `difflib`).

## One Concrete Example

```bash
# 1. two files, side by side, auto-fit to terminal width
icdiff old.json new.json

# 2. read either side from stdin via /dev/fd/N
icdiff <(curl -s https://api.example.com/v1/users/42) users.json

# 3. force a column width (useful piping into less -SR)
icdiff --cols=200 a.py b.py | less -SR

# 4. show only changed lines (drop matching context)
icdiff --line-numbers --no-headers --show-all-spaces a.txt b.txt

# 5. drop in as git's diff tool — once in git config
git config --global icdiff.options '--highlight --line-numbers'
git config --global pager.icdiff false
git icdiff HEAD~5 -- src/         # uses git-icdiff wrapper
git icdiff --staged                # review staged changes side-by-side

# 6. recursively diff two directories (one screen per file)
icdiff --recursive build-old/ build-new/ | less -R
```

## Why This Over `diff` / `delta` / `diff-so-fancy`

| ask                                              | answer                                            |
| ------------------------------------------------ | ------------------------------------------------- |
| side-by-side diff of two arbitrary files         | `icdiff`                                          |
| pretty up `git diff` (unified, inline)           | [`delta`](../delta/) or [`diff-so-fancy`](../diff-so-fancy/) |
| see *which characters* changed inside a line     | `icdiff` (bolds them inline)                      |
| works on a 1996 SunOS box with system Python     | `icdiff` (one script, stdlib only)                |
| hex / binary diff with structure                 | not `icdiff` — use `vbindiff` / `bsdiff`          |
| three-way merge view                             | not `icdiff` — use [`mergiraf`](../mergiraf/) / `vimdiff` |

`delta` and `icdiff` solve different shapes of the same problem:
`delta` is the *prettifier in front of `git diff`* (unified
output, syntax highlighting, hunk navigation); `icdiff` is the
*standalone two-file side-by-side viewer*. They compose — keep
`delta` as the default git pager and reach for `icdiff` when the
question is "let me see these two files next to each other".

## Caveats

- Side-by-side mode needs ~160 columns of terminal width to be
  comfortable. On an 80-column terminal `icdiff` will wrap
  awkwardly; either widen the terminal or fall back to `diff`
  for narrow contexts.
- Pure-Python performance is fine for source-file-sized inputs
  (tens of thousands of lines) but a 1-million-line log diff
  will be measurably slower than GNU `diff`. Pre-filter with
  `head` / `tail` / `grep` for huge files.
- The intra-line highlighting uses the same `difflib`
  ratcheting that CPython's `difflib.SequenceMatcher` uses;
  pathological inputs (lines with thousands of small differences)
  can produce surprising character-level groupings. Use
  `--whole-file` to fall back to whole-line diff in that case.
- No syntax highlighting (the differ is language-agnostic). If
  you want Python / Rust / Go syntax colors *and* side-by-side,
  pipe icdiff output through `bat` is not the right answer —
  reach for `delta --side-by-side` instead.
- Project is in long-term maintenance mode (steady bugfix
  releases since 2010); the algorithm is settled and the script
  is small enough that the lack of churn is a feature, not a
  warning.
