# fd

> **`find(1)` clone with sane defaults, regex matching, parallel
> traversal, and `.gitignore` awareness** — a single Rust binary
> that walks a directory tree and prints matching paths, with a
> command line that fits on one line. Pinned to **v10.4.2**
> (commit `7027d45303b412be6fa9c09d689cc6276748fb38`,
> [LICENSE-MIT](https://github.com/sharkdp/fd/blob/master/LICENSE-MIT),
> dual MIT / Apache-2.0).

Source: <https://github.com/sharkdp/fd>

## TL;DR

`fd PATTERN` is what you wanted `find . -name '*PATTERN*'` to be.
Pattern is a regex by default (no shell-glob escaping), case is
smart (lower-only ⇒ insensitive, mixed ⇒ sensitive), `.git/`
and anything in your `.gitignore` are skipped automatically,
hidden files are skipped unless you ask, and the walk is
parallelised across cores. Output is colourised by file type
when stdout is a TTY and plain when piped, so `fd … | xargs …`
just works. The `-x` / `-X` flags run a command per result (or
batched), giving you `find -exec` ergonomics without the
trailing `\;` ritual.

## Install

```bash
# Homebrew (macOS / Linux)
brew install fd

# Cargo
cargo install --locked fd-find

# Linux package managers
# Arch: pacman -S fd
# Debian / Ubuntu: apt install fd-find  # binary is `fdfind`
# Fedora: dnf install fd-find
# Nix: nix-env -iA nixpkgs.fd

# Windows
# scoop install fd
# choco install fd
# winget install sharkdp.fd

# from a release tarball (any OS)
curl -Lo fd.tar.gz "https://github.com/sharkdp/fd/releases/download/v10.4.2/fd-v10.4.2-aarch64-apple-darwin.tar.gz"
tar xf fd.tar.gz --strip-components=1
sudo install fd /usr/local/bin/

# verify
fd --version    # fd 10.4.2
```

On Debian / Ubuntu the binary ships as `fdfind` (collision with
the older `fd` package); alias it (`alias fd=fdfind`) or symlink
into `~/.local/bin/`.

## License

Dual MIT / Apache-2.0 — see
[LICENSE-MIT](https://github.com/sharkdp/fd/blob/master/LICENSE-MIT)
and [LICENSE-APACHE](https://github.com/sharkdp/fd/blob/master/LICENSE-APACHE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. find files whose name contains "config" anywhere under cwd
fd config

# 2. filter by extension (no leading dot needed)
fd -e rs -e toml         # all *.rs and *.toml files

# 3. include hidden files + ignored files (full walk)
fd -HI '\.env$'

# 4. find directories named "node_modules" and delete them
fd -t d node_modules -X rm -rf

# 5. find big files and pipe through xargs / batch exec
fd -t f -S +50M -x ls -lh

# 6. pipe results into another tool (auto-detects non-TTY,
#    drops colour, one path per line)
fd -e py | xargs wc -l | sort -n

# 7. case-sensitive search by mixing cases in the pattern
fd Readme                # smart-case: matches README.md, readme.txt
fd ReadMe                # case-sensitive: only ReadMe.md

# 8. follow symlinks and search outside the repo root
fd -L --search-path /etc -e conf nginx
```

## Niche It Fills

**The `.gitignore`-aware path locator.** When you are inside a
project and ask "where is the file matching X", you almost
never want hits inside `node_modules/`, `target/`, `.venv/`, or
`dist/`. `fd` honours the same ignore stack [`ripgrep`](../ripgrep/)
uses (`.gitignore`, `.ignore`, `.fdignore`) by default, so the
result list is the project's working set, not its build output.
The same one-line shape works as a `find` substitute outside a
repo (with `-HI` to walk everything).

## Why use it

Three things `fd` does that `find` does not, that pay back the
switching cost:

1. **Pattern is regex, not glob, and goes first.** `fd foo`
   searches for `foo` as a regex anywhere in the path; you do
   not need `-name '*foo*'`, you do not need to quote the glob,
   and you do not need to remember which flags come before the
   path. The order is `fd PATTERN [PATH]`, with both optional
   (no args ⇒ list everything under cwd, like `find . -type f`).
2. **`.gitignore` honoured by default.** The walker uses the
   same `ignore` crate `ripgrep` uses, so `fd` results inside
   a repo are never polluted with `node_modules/` or
   `target/debug/` noise. Override per-call with `-I` (include
   ignored), `-H` (include hidden), or `-u` (≡ `-HI`).
3. **`-x` / `-X` are first-class.** `fd -e log -x gzip` runs
   `gzip` per file in parallel across cores; `fd -e log -X tar
   czf logs.tgz` batches all results into one `tar` invocation
   via `{}` substitution. No `\;` vs `+` to remember, no
   `xargs -0 -I {}` boilerplate to type.

For an LLM-CLI workflow where you are packing a context with
[`files-to-prompt`](../files-to-prompt/) or
[`repomix`](../repomix/), `fd -e ts -e tsx src/ | xargs
files-to-prompt` gives you a precise, ignore-aware file list
the packer can consume without re-walking the tree itself.

## Vs Already Cataloged

- **Vs [`ripgrep`](../ripgrep/):** orthogonal — `ripgrep`
  searches inside files (greps content), `fd` searches across
  files (greps paths). They share the `ignore`-crate walker, so
  the set of files `rg PATTERN` reads is the same set `fd`
  prints; pair them as `fd -e rs -x rg --no-heading TODO {}` to
  scope a content search to a path filter.
- **Vs [`yazi`](../yazi/):** orthogonal — `yazi` is an
  interactive TUI file manager (browse, preview, select, act);
  `fd` is a non-interactive locator for shell pipelines and
  scripts. `yazi` even uses `fd`-shaped logic internally for
  its find-as-you-type.
- **Vs [`broot`](../broot/) (not cataloged) / `tre` (not
  cataloged):** orthogonal — those render a navigable tree
  view; `fd` emits a flat list of matching paths. Use a tree
  viewer to *understand* a directory's shape, use `fd` to
  *enumerate* matching paths for `xargs` / `-x`.
- **Vs `find` (POSIX):** `find` wins on portability (every Unix
  has it, no install) and on niche predicates `fd` does not
  expose (`-newer`, `-perm +x`, complex `-and` / `-or` trees,
  `-printf` formatting). `fd` wins on every common case: search
  by name, by extension, by type, with ignore awareness, in
  parallel, with `-x`. Keep `find` for shell scripts that must
  ship without dependencies.

## Caveats

- **`.gitignore` skip is silent.** A file under `dist/` that
  exists on disk will not appear in `fd dist` results inside a
  repo; new contributors are reliably surprised. Reach for `-I`
  (or `-u` for hidden + ignored) when chasing "why isn't `fd`
  finding it".
- **Regex syntax is Rust's `regex` crate, not POSIX.** `fd
  'foo|bar'` works as alternation as expected, but
  backreferences and lookaround are unsupported by design. For
  glob shapes use `-g 'foo*.txt'` (explicit glob mode).
- **Debian / Ubuntu rename to `fdfind`.** Same reason as
  [`bat`](../bat/) → `batcat`; alias or symlink in CI images
  that target multiple OSes, or your scripts break on those
  distros.
- **`-x` per-result invocations are parallel by default.** If
  the command being launched is itself non-deterministic in
  parallel (e.g. appending to a shared file), pin with `-j 1`.
  Use `-X` (capital) when you want exactly one batched call
  with all paths substituted in `{}`.
- **No depth-first ordering by default.** Output order is the
  walker's, which for a parallel walk is non-deterministic.
  Pipe through `sort` if your downstream consumer needs stable
  ordering across runs.
- **Symlinks are not followed unless `-L`.** A symlinked
  `node_modules` outside the repo will be invisible to default
  `fd`; pass `-L` to follow, with the usual cycle-risk caveat.
