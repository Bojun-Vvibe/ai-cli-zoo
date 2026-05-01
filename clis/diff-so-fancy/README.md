# diff-so-fancy

## What it does
A **post-processor for `git diff` output** that turns the default
`+`/`-` line dump into a more legible review format: file headers
become single bold lines instead of the four-line `diff --git ... index ... --- a/... +++ b/...` block, the leading `+`/`-` markers
are stripped (color carries the change semantics so the actual code
column stays aligned), unchanged whitespace at line ends is dimmed,
and intra-line word changes are highlighted so a one-character rename
no longer reads as a wholly new line. Plumbs in by setting
`core.pager` (or the `[pager] diff = / show =` keys) to `diff-so-fancy
| less --tabs=4 -RFX`, so every `git diff` / `git show` / `git log
-p` / `git stash show -p` invocation across every repo gets the
treatment with no per-command flag.

## Why it's interesting
Different shape from `delta` (Rust, syntax-highlights the code
itself, supports side-by-side, line numbers, file navigation — much
heavier dependency, much fancier output, and a different
configuration surface), from `diff-highlight` (the Perl script
shipped under `contrib/` in git itself — same word-highlight idea,
but no header collapsing, no whitespace handling, much plainer
output), from `git diff --color-words` (forces *only* word-level
output and loses the line context), and from `git diff --word-diff`
(inline `[-old-]{+new+}` markers — readable in plain text logs, less
so as a daily review pager). diff-so-fancy is the *zero-dependency
Perl-shell post-processor that you set as your git pager once and
forget* shape: pick it specifically when you want a clearly better
default git diff without installing a Rust binary, without learning
a `delta`-style config block, and without giving up syntax in the
code column (it leaves the source untouched — the upgrade is purely
in the chrome around it). Do **not** pick it if you specifically
want syntax-highlighted code, side-by-side view, or 24-bit themed
output (use `delta`); if you need plain-text-friendly diffs for
copy-paste into a review (`git diff --word-diff=plain`); or in
environments without Perl on `$PATH`.

## Niche category
Pure-shell git-diff post-processor — collapses headers, strips
`+`/`-` markers, highlights intra-line word changes; no Rust, no
config DSL, just a pager.

## Repo
https://github.com/so-fancy/diff-so-fancy

## Version pinned
`v1.4.10` (latest tagged release per
`gh api /repos/so-fancy/diff-so-fancy/releases/latest`)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install diff-so-fancy

# npm (any platform with Node)
npm install -g diff-so-fancy

# Manual (Perl is the only runtime dep)
curl -L https://raw.githubusercontent.com/so-fancy/diff-so-fancy/v1.4.10/diff-so-fancy \
  -o /usr/local/bin/diff-so-fancy
chmod +x /usr/local/bin/diff-so-fancy
```

## Usage examples
```sh
# Wire up as the default git pager for every diff / show / log -p
git config --global core.pager "diff-so-fancy | less --tabs=4 -RFX"
git config --global interactive.diffFilter "diff-so-fancy --patch"

# Recommended color tweaks for a more legible default theme
git config --global color.ui true
git config --global color.diff-highlight.oldNormal    "red bold"
git config --global color.diff-highlight.oldHighlight "red bold 52"
git config --global color.diff-highlight.newNormal    "green bold"
git config --global color.diff-highlight.newHighlight "green bold 22"
git config --global color.diff.meta       "11"
git config --global color.diff.frag       "magenta bold"
git config --global color.diff.func       "146 bold"
git config --global color.diff.commit     "yellow bold"
git config --global color.diff.old        "red bold"
git config --global color.diff.new        "green bold"
git config --global color.diff.whitespace "red reverse"

# One-shot, without changing global config
git diff --color | diff-so-fancy | less -RFX

# Strip the leading +/- markers off any unified-diff stream
cat patch.diff | diff-so-fancy
```
