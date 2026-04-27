# sd

> **Intuitive find-and-replace CLI — a `sed(1)` alternative** in
> Rust with PCRE-style regex by default, literal strings without
> escaping, in-place edit with `-i`, and stdin → stdout pipe
> behaviour. Pinned to **v1.1.0** (commit
> `44febdf86343c653255ae4f20f5e1882dad6be17`,
> [LICENSE](https://github.com/chmln/sd/blob/master/LICENSE),
> MIT).

Source: <https://github.com/chmln/sd>

## TL;DR

`sed -i 's/foo/bar/g' file` is the muscle memory; `sd foo bar
file` is the same thing without the BRE / ERE / GNU-vs-BSD
landmines. `sd` takes two positional arguments — *find* and
*replace* — and reads from stdin or a list of files, writing back
in place when given paths and to stdout when given a pipe. The
*find* string is a real PCRE-style regex (`\d`, `\b`, lookarounds,
named groups, lazy quantifiers all work as you would expect from
`grep -P`), `$1`-`$9` and `${name}` are the backreference syntax
in *replace*, and a `--literal` / `-F` flag turns the whole thing
into a fast plain-string substitution when you do not want any
characters to be magic. The killer ergonomic detail: no escaping
of `/` (separators are not character-based), no `\(...\)` BRE
groups, no `-E` flag to "enable real regex", no awkward
`s|a/b|c/d|` workaround — the binary is one job and one syntax.
Cross-platform (Linux, macOS, FreeBSD, Windows).

## Install

```bash
# Homebrew (macOS / Linux)
brew install sd

# Cargo
cargo install --locked sd

# Linux package managers
# Arch: pacman -S sd
# Debian / Ubuntu: apt install sd          # 23.04+ ; binary may collide with systemd `sd`
# Fedora: dnf install sd
# Nix: nix-env -iA nixpkgs.sd

# Windows
# scoop install sd
# choco install sd
# winget install chmln.sd

# release tarball (any OS / arch)
curl -Lo sd.tar.gz "https://github.com/chmln/sd/releases/download/v1.1.0/sd-v1.1.0-aarch64-apple-darwin.tar.gz"
tar xf sd.tar.gz --strip-components=1 && sudo install sd /usr/local/bin/

# verify
sd --version    # sd 1.1.0
```

A handful of distros ship a *different* binary called `sd` (most
visibly the systemd toolchain on a couple of immutable distros);
if `sd --version` reports anything other than `sd <semver>`, install
via `cargo` or use the explicit `chmln.sd` package id and adjust
`$PATH`.

## License

MIT — see
[LICENSE](https://github.com/chmln/sd/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. stdin → stdout, single shot (the most common shape)
echo "hello world" | sd 'world' 'sd'        # hello sd

# 2. real PCRE — no -E flag, no escaped parentheses
echo 'order_42 paid 199.99' | sd '(\w+)_(\d+) paid ([\d.]+)' '#$2 ($1) -> $$$3'
# #42 (order) -> $199.99

# 3. in-place edit on a single file
sd 'localhost:5432' 'db.internal:5432' src/config.toml

# 4. in-place across many files via fd / find
fd -e ts -e tsx . src/ -x sd 'oldName' 'newName'
# (fd's -x runs sd on each match in parallel)

# 5. literal mode (no regex magic) — useful for paths / regex chars
sd -F './src/foo.ts' './src/bar.ts' Cargo.toml package.json

# 6. lookbehind / lookahead (not portable in BSD sed)
sd '(?<=version = ")\d+\.\d+\.\d+(?=")' '1.2.3' Cargo.toml

# 7. preview before writing — pipe through bat to see the diff shape
sd 'foo' 'bar' src/main.rs --preview | bat -l rust

# 8. multi-line regex (the `s` flag in `(?s)` makes `.` match `\n`)
sd '(?s)BEGIN.*?END' 'TRUNCATED' README.md
```

## Niche It Fills

**The find-and-replace primitive.** `sed` is a stream editor with
addressing, hold space, branches, labels, and multi-command
programs — it is a small Turing-complete language with a half-
dozen incompatible dialects. 95% of the times you reach for `sed`
you wanted exactly one thing: substitute *find* with *replace*,
optionally in place. `sd` is that one thing, with one regex
dialect (PCRE), one substitution syntax (`$1`), one in-place flag
behaviour (positional file args), and a `--literal` escape hatch.
For everything `sed` can do that this does not — addressing
(`/foo/d`), hold space, multi-command pipelines, branches — keep
`sed` (or use `awk`); for the substitute case, `sd` removes the
escaping tax.

## Why use it

Three things `sd` does that `sed` does not, that pay back the
muscle-memory cost:

1. **PCRE by default — same regex you wrote in your editor.**
   `\d`, `\w`, `\b`, `\s`, lazy quantifiers (`.*?`), lookarounds
   (`(?<=…)`, `(?!…)`), named groups (`(?P<n>…)`), `(?s)` /
   `(?i)` flag groups all work without `-E` / `-P` / `--perl`
   incantations. No more "this regex works in my IDE but not in
   `sed`" — it works in `sd`.
2. **No separator escaping.** `sed` uses one character (typically
   `/`) as the separator and forces you to escape every literal
   `/` in the pattern (`s/\/usr\/local\/bin/\/opt\/bin/`); `sd`
   takes *find* and *replace* as positional arguments, so paths,
   URLs, and JSON pointers are written literally
   (`sd '/usr/local/bin' '/opt/bin' file`). Saves real time on
   path / URL substitutions.
3. **In-place edit is the unsurprising default.** If you pass
   file paths, `sd` writes back to those files; if you pipe in
   stdin, `sd` writes to stdout. There is no `-i ''` BSD vs GNU
   incompatibility (BSD sed requires an empty backup-suffix arg
   that GNU sed treats as a backup-suffix; the same script
   silently does the wrong thing on the other platform). One
   binary, one behaviour, both Macs and Linux.

For an LLM-CLI workflow `sd` is the fix-step after a model
suggests a rename: `fd -e py | xargs sd 'old_api_call' 'new_api_call'`
applies a clean, lookaround-aware rename across the tree without a
text editor.

## Vs Already Cataloged

- **Vs [`ast-grep`](../ast-grep/):** orthogonal — `ast-grep`
  understands the AST of the language it is rewriting, so
  `ast-grep -p 'console.log($A)' -r 'logger.info($A)' --lang ts`
  rewrites only real call expressions, never matches inside
  strings or comments, and respects scoping. `sd` is purely
  textual — fast, language-agnostic, but happy to rewrite
  matches inside string literals or comments. Use `sd` for
  textual / config / log substitutions; use `ast-grep` for
  language-aware code refactors where false positives matter.
- **Vs [`ripgrep`](../ripgrep/) (`rg`):** complementary.
  `rg foo` finds the matches; `sd foo bar file` rewrites them.
  The two tools compose: `rg -l 'oldFlag' | xargs sd 'oldFlag'
  'newFlag'` is the canonical "find every file that uses X, then
  rewrite X" pipeline. They share a regex dialect and `.gitignore`
  awareness comes from `rg`'s side.
- **Vs `sed` / `perl -pe` (POSIX / classic):** `sed` is the right
  answer when you need addressing (`/foo/d`, `1,10s/.../.../`),
  hold-space tricks (`N`, `H`, `G`, `D`), or multi-command
  programs (`-e`); `perl -pe` is the right answer when the
  substitution body needs to call out to Perl (arithmetic,
  capture transforms, `eval`). `sd` is the right answer for the
  80% case: substitute *find* with *replace*, in stdin or in
  files, with a regex you actually remember.

## Caveats

- **Substitute-only, by design.** No deletion (`/pattern/d`), no
  insertion (`i`/`a`), no addressed ranges (`1,10s/…`), no
  multiple commands (`-e`/`-f`). For those, `sed` or `awk`
  remain the right answer. (`sd 'pattern' '' file` works as a
  delete for the matched substring, but not for whole lines.)
- **No backup file flag.** Where GNU `sed -i.bak` writes
  `file.bak` next to the in-place edit, `sd` overwrites in
  place with no backup. Wrap in `git diff` / `--preview` for the
  safety net (or run on a clean working tree).
- **`--preview` shows result, not diff.** The `--preview` flag
  prints the post-substitution content to stdout instead of
  writing in place; it does not show a unified diff. Pipe
  through `diff -u <(cat file) <(sd '…' '…' --preview file)` (or
  [`delta`](../delta/)) for a real diff view before committing.
- **Multi-line matches need an opt-in flag-group.** Plain `.`
  does not span newlines; you have to prefix with `(?s)` or
  use `[\s\S]` to match across line breaks. This matches
  PCRE / Rust-regex semantics but surprises people coming from
  GNU `sed` with `N` in the script.
- **No `-z` / NUL-separated mode.** Unlike GNU `sed -z` you can
  not split the input on `\0` — for filename pipelines coming
  from `find -print0` reach back to `xargs -0 -n1 sd …` or use
  GNU `sed`.
- **Binary-name collision on a few distros.** A non-Rust `sd` (a
  systemd helper) ships on a handful of immutable distros; check
  `sd --version` after install to confirm you got
  `chmln/sd v1.x.y`.
