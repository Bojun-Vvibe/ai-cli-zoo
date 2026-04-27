# ripgrep

> **Recursively search directories for a regex pattern, fast** —
> a single Rust binary (`rg`) that walks a tree, respects
> `.gitignore` / `.ignore` / `.rgignore` / hidden-file rules by
> default, uses Rust's `regex` engine with SIMD-accelerated literal
> prefilters, and ships parallel per-file search out of the box.
> Pinned to **v15.1.0** (commit
> `af60c2de9d85e7f3d81c78601669468cf02dabab`,
> [LICENSE-MIT](https://github.com/BurntSushi/ripgrep/blob/master/LICENSE-MIT),
> dual MIT / Unlicense).

Source: <https://github.com/BurntSushi/ripgrep>

## TL;DR

`rg` is the search tool you reach for when `grep -r` is taking
seconds-to-minutes on a large repo, when you keep matching inside
`node_modules/` / `target/` / `.git/`, and when the regex has to
straddle "literal substring fast-path" and "real PCRE-ish
features." Same shape as `grep` (`rg PATTERN [PATH]`), but the
defaults are tuned for source code: it descends recursively from
`.` if you pass no path, skips `.gitignore`d files, skips hidden
files, skips binary files, and prints `path:line:match` with the
file path coloured and the match highlighted. The engine is Rust's
`regex` crate (linear-time guarantee, SIMD literal scan); for
backreferences and lookaround pass `-P` to switch to PCRE2. On a
typical 10k-file repo `rg pattern` finishes in well under a second
on commodity hardware, two-to-three orders of magnitude faster
than naive `grep -r`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ripgrep

# Cargo
cargo install --locked ripgrep

# Linux package managers
# Arch: pacman -S ripgrep
# Debian / Ubuntu: apt install ripgrep
# Fedora: dnf install ripgrep
# Nix: nix-env -iA nixpkgs.ripgrep

# Windows
# scoop install ripgrep
# choco install ripgrep
# winget install BurntSushi.ripgrep.MSVC

# from a release tarball (any OS)
curl -Lo rg.tar.gz "https://github.com/BurntSushi/ripgrep/releases/download/15.1.0/ripgrep-15.1.0-aarch64-apple-darwin.tar.gz"
tar xf rg.tar.gz --strip-components=1
sudo install rg /usr/local/bin/

# verify
rg --version    # ripgrep 15.1.0
```

No config required. Drop a `~/.config/ripgrep/config` and point
`RIPGREP_CONFIG_PATH` at it for persistent flags (`--smart-case`,
`--hidden`, custom `--type-add`, etc.). Shell completions ship in
the release tarball.

## License

Dual MIT / Unlicense — see
[LICENSE-MIT](https://github.com/BurntSushi/ripgrep/blob/master/LICENSE-MIT)
and [UNLICENSE](https://github.com/BurntSushi/ripgrep/blob/master/UNLICENSE).
Permissive; the Unlicense option is effectively public-domain
dedication, useful when redistributing in jurisdictions that need
one.

## One Concrete Example

```bash
# 1. recursive search from cwd, gitignore-aware, coloured
rg 'TODO\(.*\)'

# 2. case-smart (lowercase = case-insensitive, mixed = sensitive)
rg --smart-case 'parseConfig'

# 3. restrict by file type (built-in type registry)
rg -t py     'def __init__'      # Python only
rg -t rust   'unsafe fn'         # Rust only
rg --type-list                   # show all known types

# 4. limit to a glob (negate with `!`)
rg -g '*.toml' '\[dependencies\]'
rg -g '!**/test/**' 'eprintln!'

# 5. show N lines of context around each hit
rg -C 2 'panic!'                  # 2 before + 2 after
rg -A 5 'fn main'                 # 5 lines after

# 6. count matches per file (great for "how widespread is this")
rg -c 'unwrap\(\)' --sort path

# 7. emit JSON for downstream tooling (one event per match / context)
rg --json 'TODO' src/ | jq -r 'select(.type=="match") | .data.path.text'

# 8. multi-line mode — pattern can match across newlines
rg -U --multiline-dotall 'fn \w+\([^)]*\)\s*\{[^}]*panic'

# 9. PCRE2 features (lookaround, backreferences)
rg -P '(?<!\.)\bclient\.connect\b'

# 10. search inside compressed / archived files
rg -z 'kernel: oom-killer' /var/log/

# 11. replace preview to stdout (does NOT modify files)
rg 'OldName' --replace 'NewName' src/

# 12. as the input for fzf in an editor-jump pipeline
rg --line-number --no-heading --color=always '' \
  | fzf --ansi --delimiter ':' --preview 'bat --color=always --line-range {2}: {1}'
```

## Niche It Fills

**Default-correct recursive code search.** For "find this string
in this repo," `grep -r` requires you to remember to skip
`node_modules`, `.git`, build artefacts, binary blobs, and to
add `--include` for the right file types — every invocation is a
bespoke flag string. `rg` makes the right choices the default
(gitignore-aware, hidden-file-aware, binary-skip, parallel-walk)
so the bare command does what you want. The flag surface is then
about *narrowing further* (`-t`, `-g`, `--no-ignore`) instead of
*excluding noise*, which is the inverted polarity that pays back
the muscle-memory switch from `grep`.

## Why use it

Three things `rg` does that `grep -r` does not, that pay back the
switching cost:

1. **`.gitignore` / `.ignore` aware by default.** A fresh
   `rg pattern` in a Node monorepo skips `node_modules/`, in a
   Rust workspace skips `target/`, in a Go project skips `vendor/`
   if those are ignored — without you adding `--exclude-dir`. To
   force-include, `--no-ignore` (one flag) or
   `-uuu` (all the way) lifts every default exclusion.
2. **Parallel per-file scan with SIMD literal prefilter.** `rg`
   walks the directory tree concurrently, picks the longest
   literal substring out of your regex, scans for it with
   vectorised byte search (memchr / Aho-Corasick), and only
   evaluates the full regex on the few candidate windows. On a
   warm filesystem cache the result is wall-clock time roughly
   linear in *match candidates*, not file size.
3. **Built-in file-type registry.** `rg -t py PATTERN` /
   `rg -t rust PATTERN` /
   `rg -t web PATTERN` (HTML + CSS + JS + TS + …) hides the
   "construct a glob list of every Python extension" step;
   `rg --type-add 'proto:*.proto'` extends the registry per-call
   or per-config.

For an LLM-CLI workflow where an agent has to grep a large
codebase before deciding what to touch, `rg --json` is a clean
structured input — line / column / submatch positions in JSON
events stream directly into a tool-call result without parsing
text-shaped output.

## Vs Already Cataloged

- **Vs [`ast-grep`](../ast-grep/):** orthogonal — `ast-grep`
  matches by syntax tree pattern (`$$$F.then($$$)` finds promise
  chains across formatting differences), `rg` matches by regex
  on bytes. Use `ast-grep` when you want "find this code shape"
  (refactor candidates, lint patterns); use `rg` when you want
  "find this string / token / comment" (TODOs, error messages,
  config keys, log lines). Many workflows pipe `rg` first as a
  cheap pre-filter into `ast-grep` for the structural pass.
- **Vs [`grep-ast`](../grep-ast/):** orthogonal — `grep-ast`
  prints whole syntactic units (function bodies, class blocks)
  containing a match, designed for LLM-context packing; `rg`
  prints lines or N lines of context. Use `grep-ast` when you
  want "the whole function around this match" for an LLM prompt;
  use `rg` for everything else.
- **Vs [`probe`](../probe/) / [`seagoat`](../seagoat/):** those
  are AI-flavoured semantic code search (embeddings, LLM ranking)
  for "find code that does X conceptually." `rg` is exact-string /
  regex search. Different question shapes; in practice
  semantic-search tools call `rg` underneath as a filter.
- **Vs `grep` / `ag` / `ack` (POSIX / not cataloged):** `grep` is
  POSIX, everywhere, and right for tiny one-off searches in known
  small inputs. `ag` (the_silver_searcher) was the previous
  generation of "fast recursive grep" — `rg` superseded it on
  speed (parallel walk + SIMD prefilter) and feature surface
  (PCRE2, multi-line, JSON output, type registry); `ag` is now
  in maintenance mode. `ack` is Perl-based and easier to extend
  in Perl but slower and less ubiquitous.

## Caveats

- **Default-skip surprises.** `rg` will *not* see files in
  `.gitignore`, hidden files, binary files, or files larger than
  no limit (size limits are off by default but symlinks-out-of-
  tree are skipped). When a search "finds nothing" and you
  expected a hit, retry with `-uuu` (no ignore, hidden,
  binary). The `--debug` flag prints which rules excluded which
  paths.
- **Regex engine is not Perl-compatible by default.** Rust
  `regex` is linear-time and rejects backreferences and arbitrary
  lookaround; if your pattern uses `(?<=foo)` or `\1` you must
  pass `-P` for the PCRE2 backend (slower, no linear guarantee,
  but full Perl features). Most patterns do not need this.
- **Output format changes between TTY and pipe.** TTY default is
  grouped (`path:` header + `line:match` lines + blank separator);
  pipe default is flat (`path:line:match` per line). Scripts
  depending on layout should pin `--no-heading --line-number`
  explicitly rather than rely on auto-detection.
- **`--replace` does not edit files.** It prints what the
  substitution would look like to stdout; for in-place rewrite
  use [`sd`](https://github.com/chmln/sd) or `sed -i` after
  filtering candidate files with `rg -l`. This is intentional —
  `rg` is a search tool, and a "search and replace" tool that
  silently rewrote your tree on a typo would be a footgun.
- **Symlink handling is conservative.** By default `rg` does not
  follow symlinks (avoids loops + escaping the tree); pass `-L`
  to follow. In a worktree with bind-mounted vendored code this
  is the flag to remember.
- **Encoding detection is best-effort.** `rg` assumes UTF-8 with
  a fallback to byte-mode; for UTF-16 / Latin-1 / GBK source
  files pass `-E utf-16` (or whichever) to force the decode, or
  matches inside non-ASCII strings will silently miss.
