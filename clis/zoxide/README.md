# zoxide

> **A smarter `cd` that learns your habits** — a single Rust
> binary that records every directory you `cd` into, scores them
> by *frecency* (frequency × recency), and exposes one new
> command (`z`) that jumps to the highest-scoring directory
> matching a substring. Pinned to **v0.9.9** (commit
> `9cdc6aa3740b4d8a9d62406c99e84c5de49645e9`,
> [LICENSE](https://github.com/ajeetdsouza/zoxide/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ajeetdsouza/zoxide>

## TL;DR

`zoxide` is the replacement for `cd ~/some/long/nested/project/path`
when you have repeated that incantation enough times that your
shell history is mostly that one line. After installing the shell
hook (one line in `.zshrc` / `.bashrc` / `.config/fish/config.fish`),
every successful `cd` is recorded into a tiny SQLite-style
database; from then on `z proj` jumps to the most-frecent directory
whose path contains `proj`. Multiple substrings narrow further:
`z proj api` jumps to the directory containing both `proj` and
`api` somewhere in the path. There is also `zi` (interactive picker
via `fzf`) and `z -` (jump to previous directory). The whole tool
is one ~2 MB binary plus one shell-init line; the database lives
in `~/.local/share/zoxide/db.zo`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install zoxide

# Cargo
cargo install --locked zoxide

# Linux package managers
# Arch: pacman -S zoxide
# Debian / Ubuntu: apt install zoxide
# Fedora: dnf install zoxide
# Nix: nix-env -iA nixpkgs.zoxide

# Windows
# scoop install zoxide
# choco install zoxide
# winget install ajeetdsouza.zoxide

# from a release tarball (any OS)
curl -Lo zoxide.tar.gz "https://github.com/ajeetdsouza/zoxide/releases/download/v0.9.9/zoxide-0.9.9-aarch64-apple-darwin.tar.gz"
tar xf zoxide.tar.gz
sudo install zoxide /usr/local/bin/

# verify
zoxide --version    # zoxide 0.9.9

# Add to your shell rc — pick one:
# ~/.zshrc
eval "$(zoxide init zsh)"
# ~/.bashrc
eval "$(zoxide init bash)"
# ~/.config/fish/config.fish
zoxide init fish | source
# nushell config.nu
zoxide init nushell | save -f ~/.zoxide.nu
```

After the shell hook is installed, just `cd` around normally for a
few days — `zoxide` builds the database in the background. Then
start using `z <substring>` instead of `cd <long path>`. To
preserve your existing muscle memory, many users `alias cd=z` once
they trust it; `z` falls back to literal `cd` for any path that
exists exactly.

## License

MIT — see
[LICENSE](https://github.com/ajeetdsouza/zoxide/blob/main/LICENSE).
Permissive, no attribution required for binaries; redistribute
freely.

## One Concrete Example

```bash
# 1. jump to the highest-frecency directory matching "proj"
z proj

# 2. multi-substring narrowing — match all of these in the path
z my api v2     # → ~/work/myorg/api-service/v2-rewrite

# 3. interactive picker (requires fzf in PATH)
zi proj         # arrow-key through ranked candidates

# 4. jump to previous directory (like `cd -`)
z -

# 5. inspect the frecency database
zoxide query --list --score
# 142.3  /Users/me/work/myorg/api-service
#  98.1  /Users/me/work/myorg/web-app
#  41.2  /Users/me/Projects/dotfiles
#  ...

# 6. add a directory without cd-ing into it (warm-start the db)
zoxide add ~/big/repo/i/will/use/often

# 7. remove a stale entry (deleted directory still scored)
zoxide remove ~/old/path/that/no/longer/exists

# 8. seed zoxide from your existing autojump / z (bash) database
zoxide import --from autojump ~/.local/share/autojump/autojump.txt
zoxide import --from z       ~/.z

# 9. drop-in cd replacement (in your shell rc)
alias cd=z      # now `cd proj` does the smart thing
                # and `cd /absolute/path` still cd's normally
```

## Niche It Fills

**Directory navigation as a learned-prior, not a typed-prefix.**
The default `cd` workflow assumes you remember and type the path
every time, then offsets the cost with `cd -`, tab completion, and
shell history search. `zoxide` inverts the model: the shell
remembers *for you* which directories you actually visit, and
exposes a one-letter command that fuzzy-matches them in
frecency-ranked order. After ~a week of installation the typical
"go to a working directory" cost drops from ~12 keystrokes (or a
ctrl-R search) to ~5 (`z foo<enter>`).

## Why use it

Three things `zoxide` does that `cd` does not, that pay back the
switching cost:

1. **Frecency scoring, not just frequency or recency.** A
   directory you visit constantly stays high; one you visited
   often last year but not this week decays; one you visited five
   times yesterday rises fast. The score is recomputed at every
   `z` call, so the ranking adapts as your project focus shifts —
   no manual aliasing of "current project" required.
2. **Substring narrow + auto-disambiguation.** `z api` is enough
   when one path matches; when many do, add another substring
   (`z api v2`) until exactly one wins, or pop the `fzf` picker
   with `zi`. Compared to maintaining a list of `alias work='cd
   ~/work/myorg/api-service/v2'` entries, it's the same ergonomics
   without the manual upkeep.
3. **Pure shell hook + tiny binary, no daemon.** The shell hook
   is a single trap on the `chpwd`/`PROMPT_COMMAND` event that
   appends the new directory to the database; the binary is
   ~2 MB, the database is a flat file in `~/.local/share/zoxide/`.
   Nothing background, nothing networked, nothing privileged —
   safe to install on a shared box.

For an LLM-CLI workflow where an agent needs to navigate to a
project root, `zoxide query proj` returns the absolute path of
the top-scored match in plain text, suitable for `cd "$(zoxide
query proj)"` inside a tool call without the agent having to
remember (or be told) the full path.

## Vs Already Cataloged

- **Vs [`atuin`](../atuin/):** orthogonal — `atuin` replaces
  *shell history* (every command you ran) with a synced SQLite
  store + fuzzy search; `zoxide` replaces *directory history*
  (every directory you cd'd into) with a frecency-ranked store.
  They cover non-overlapping verbs: `atuin` answers "what was
  that long command I ran last week", `zoxide` answers "take me
  back to that project". Most heavy CLI users install both.
- **Vs [`yazi`](../yazi/):** orthogonal — `yazi` is a TUI file
  manager (visual two-pane navigation, previews, batch ops);
  `zoxide` is a shell-level jump-to-directory shortcut. Use
  `yazi` when you want to *look at* a tree and operate on files;
  use `zoxide` when you already know which directory you want to
  *be in* and just want to get there fast.
- **Vs `cd -` / `pushd`/`popd` (POSIX):** `cd -` toggles between
  exactly two directories; `pushd`/`popd` maintain an explicit
  stack you have to manage. `zoxide` covers the much more common
  case of "jump to one of the ~20 directories I actually work in
  this quarter" without manual stack discipline.
- **Vs `autojump` / `fasd` / `z` (the original bash script) (not
  cataloged):** the closest peers — same frecency-jump idea,
  predecessors by ~10 years. `zoxide` is faster (Rust binary vs
  Python / awk script), supports more shells (bash / zsh / fish /
  nushell / pwsh / elvish out of the box), has fewer footguns
  (no `awk` parsing of paths with spaces), and ships an
  `--import` path from each of the older tools so your existing
  database transfers.

## Caveats

- **Two-keystroke fork: `z` vs `zi`.** `z foo` jumps directly to
  the top match (silent if unambiguous, surprising if not); `zi
  foo` always opens the `fzf` picker even on a single match. New
  users sometimes alias `z=zi` and lose the speed win — pick a
  policy and stick with it.
- **Database is per-user, not per-machine-shared.** No built-in
  sync between laptops; if you want that, version-control
  `~/.local/share/zoxide/db.zo` in your dotfiles, or use a tool
  like `chezmoi` to template it. Shared accounts (e.g. a `dev`
  user on a server) will pollute each other's frecency.
- **No path-validation on score time.** Deleted directories stay
  in the database with their old score until you `z` to them
  (and they fail) or run `zoxide remove`. The maintenance
  command `zoxide query --list` makes the cruft visible.
- **Substring match is on the full path.** `z api` matches both
  `~/work/api-service` and `~/notes/apiology`; the frecency
  ranking decides. For paths that share words across unrelated
  projects, narrow with a second substring or use `zi` and pick
  visually.
- **Shell-init eval is non-trivial.** `eval "$(zoxide init zsh)"`
  defines `z`, `zi`, the `chpwd` hook, and (optionally) `cd`
  alias; if your `.zshrc` overrides `chpwd_functions` or `cd`
  itself elsewhere, the hook can silently lose updates. The
  `--no-cmd` and `--cmd` flags to `init` let you rename or
  disable each piece if it conflicts.
- **Requires a successful `cd` to learn.** If you only ever
  navigate via file managers, IDEs that `cd` in a separate
  process, or `tmux`'s `default-path`, the database stays empty.
  Use `zoxide add <path>` in your editor's `cd` hook, or accept
  that the win only materialises in shells where you actually
  type `cd`.
