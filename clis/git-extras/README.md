# git-extras

> **70+ git subcommands** that fill the gaps GNU git leaves —
> `git summary`, `git changelog`, `git effort`, `git ignore`,
> `git undo`, `git delete-merged-branches`, `git fork`, and
> dozens more — all installed as `git-*` shell scripts on
> `PATH` so each becomes a first-class `git <verb>`.
> Pinned to **v7.4.0** (SPDX: `MIT`,
> [LICENSE](https://github.com/tj/git-extras/blob/master/LICENSE)).

Source: <https://github.com/tj/git-extras>

## TL;DR

`git-extras` is a curated collection of small portable shell
scripts that ship as `git-summary`, `git-changelog`,
`git-effort`, etc. Because git auto-discovers any `git-<verb>`
on `PATH` as a subcommand, installing the package gives you
~70 new `git <verb>` calls without touching `~/.gitconfig` or
adding aliases. Each one is tens to hundreds of lines of
`sh`, auditable, and replaceable.

## Install

```bash
# Homebrew (macOS / Linux)
brew install git-extras

# apt (Debian / Ubuntu)
sudo apt install git-extras

# from source
git clone https://github.com/tj/git-extras.git
cd git-extras && sudo make install

# verify
git extras --version    # 7.4.0
git extras              # list every subcommand it added
```

## License

MIT — see
[LICENSE](https://github.com/tj/git-extras/blob/master/LICENSE).

## One Concrete Example

```bash
# 1. who has been touching the codebase, ranked
git summary
# project  : myapp
# repo age : 4 years, 2 months
# active   : 412 days
# commits  : 3,127
# files    : 821
# authors  :
#   1841  Alice Example   58.9%
#   862   Bob Sample      27.6%
#   ...

# 2. file-by-file effort: which paths churn?
git effort --above 50    # only files with > 50 commits

# 3. autogenerate CHANGELOG.md from git tags
git changelog              # rewrites CHANGELOG.md, prepending the new section
git changelog --tag v1.4.0 # explicit tag bound

# 4. one-shot .gitignore from gitignore.io templates
git ignore-io node python macos > .gitignore

# 5. local + tracked .gitignore management
git ignore "*.log" build/

# 6. undo the last commit, keeping the changes staged
git undo            # = git reset --soft HEAD~1
git undo 3          # undo last 3 commits

# 7. clean up branches whose upstreams are gone
git delete-merged-branches
git delete-squashed-branches    # squash-merge aware

# 8. open the current repo on GitHub / GitLab in the browser
git browse

# 9. fork a GitHub repo and clone it (uses gh CLI under the hood)
git fork tj/some-repo

# 10. list all aliases defined in any of your gitconfigs
git alias
```

## Niche It Fills

**Auditable shell-script extensions to `git`** — every
`git-*` script is a small standalone file you can `cat
$(which git-summary)` and read end to end. The catalog as a
whole gives you the "I always wished git could do this in one
verb" subcommands without each one being its own install.

## Why use it

1. **One install, ~70 verbs.** No piecemeal `cargo install
   git-summary && cargo install git-effort && ...` loop.
2. **Discovery is just `git extras`.** Lists every subcommand
   the package added, with a one-line description per — your
   permanent reference.
3. **Plain shell, plain audit.** Each command is a sh / bash
   script in `/opt/homebrew/share/git-extras/` (or distro
   equivalent). No compiled binary, no plugin runtime.
4. **Convention over config.** `git changelog` reads tags,
   `git effort` reads the log, `git summary` reads HEAD —
   nothing to configure.
5. **Composable.** Output goes to stdout, exit codes are
   sensible, every command is `xargs` / `awk` / `jq`-friendly.

## Vs Already Cataloged

- **Vs [`gh`](../gh/) / [`glab`](../glab/):** Those are *forge*
  CLIs (PRs, issues, releases) talking to GitHub / GitLab APIs.
  `git-extras` is purely about the *local* repo — neither
  replaces the other; they compose (`git browse` calls them).
- **Vs [`git-cliff`](../git-cliff/):** `git cliff` is a
  template-driven Conventional-Commits changelog generator
  with a config file. `git changelog` is simpler: append a new
  section to `CHANGELOG.md` from `git log <last-tag>..HEAD`,
  no template, no config — pick `git-cliff` when you need the
  template control, `git-extras` when you don't.
- **Vs [`commitizen`](../commitizen/) / [`cocogitto`](../cocogitto/):**
  Those drive the *release* (version bump + tag + push +
  publish) with Conventional-Commits enforcement.
  `git-extras` does not enforce a commit format — it just
  exposes utilities (`git release`, `git changelog`).
- **Vs [`git-branchless`](../git-branchless/) /
  [`git-machete`](../git-machete/):** Those provide
  *opinionated workflows* on top of git (anonymous branches,
  branch-stack management). `git-extras` provides standalone
  utilities, not a workflow.
- **Vs [`tig`](../tig/) / [`gitui`](../gitui/) /
  [`lazygit`](../lazygit/):** Those are interactive TUI git
  clients. `git-extras` is purely additive subcommands for the
  CLI flow.

## Caveats

- **Shell scripts, not portable to Windows native.** Works
  under WSL, Git Bash, or Cygwin; not as native PowerShell.
- **Some commands assume `gh` is installed** (`git fork`,
  `git pr`). They fall back gracefully but the killer features
  need it.
- **`git changelog` is opinionated.** Plain commit-log dump
  per tag — no Conventional-Commits classification. Pair with
  [`git-cliff`](../git-cliff/) for that.
- **No update mechanism per command.** Updating the package
  updates all 70 — no per-script pinning. Acceptable because
  the scripts are tiny and rarely break.
