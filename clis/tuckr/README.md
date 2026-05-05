# tuckr

> **A super-powered replacement for GNU Stow** — a single static
> Rust binary that symlinks dotfiles from a per-program tree
> (`~/.dotfiles/<program>/`) into your `$HOME`, but unlike stow
> adds first-class **hooks** (per-program pre/post setup
> scripts), **secrets** (encrypted files transparently decrypted
> on `tuckr add`), **conditional groups** (`group@arch`,
> `group@laptop`), and a real status command that shows which
> dotfiles are linked, conflicting, or stale. Pinned to
> **v0.13.1** (SPDX: `GPL-3.0`,
> [LICENSE](https://github.com/RaphGL/Tuckr/blob/main/LICENSE)).

Source: <https://github.com/RaphGL/Tuckr>

## TL;DR

Stow has been the default "symlink my dotfiles" tool for two
decades, but it does exactly one thing — symlinks — and treats
hooks, conditional inclusion, and secret material as
out-of-scope. `tuckr` keeps the stow mental model (one
directory per program, symlinks into `$HOME`) and adds the four
things every dotfile repo eventually grows out-of-band shell
scripts for:

1. **Hooks** — `tuckr/Hooks/<program>/pre.sh` and `post.sh`
   run automatically on `tuckr add <program>`. Install
   `nvim` plugins, register a launchd agent, `chsh -s zsh` —
   all colocated with the dotfile, no `bootstrap.sh` god-script.
2. **Secrets** — `tuckr/Secrets/<program>/` is encrypted at
   rest (age / passphrase). `tuckr add` decrypts on link, `tuckr
   encrypt` re-seals on commit. The repo can stay public.
3. **Conditional groups** — name a directory `nvim@laptop` or
   `i3@arch` and it only links on hosts matching the tag. Same
   repo, multiple machines, no per-host `if`-tree in a Makefile.
4. **Status / diff / dry-run** — `tuckr status` shows a
   colorized table of linked / unlinked / conflicting /
   modified files. `tuckr add --dry-run` previews symlink
   changes before they touch `$HOME`. Stow's failure mode (a
   typo in your tree leaves a half-linked `$HOME`) becomes a
   solved problem.

## Install

```bash
# Cargo (any OS with a Rust toolchain)
cargo install tuckr --locked

# Arch Linux (AUR)
yay -S tuckr-bin

# from a release binary
curl -Lo tuckr.tar.gz "https://github.com/RaphGL/Tuckr/releases/download/0.13.1/tuckr-x86_64-unknown-linux-gnu.tar.gz"
tar xf tuckr.tar.gz && sudo install tuckr /usr/local/bin/

# verify
tuckr --version    # tuckr 0.13.1
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/RaphGL/Tuckr/blob/main/LICENSE).
Copyleft: derivative works (forks that ship a modified `tuckr`
binary) must publish source under a compatible license.
*Configs* you manage with `tuckr` are unaffected — GPL applies
to the tool, not to the dotfiles it symlinks.

## One Concrete Example

```bash
# 1. layout — one directory per program, mirroring $HOME
~/.dotfiles/
├── Configs/
│   ├── nvim/.config/nvim/init.lua
│   ├── zsh/.zshrc
│   └── git/.gitconfig
├── Hooks/
│   └── nvim/post.sh           # nvim --headless +PlugInstall +qall
└── Secrets/
    └── ssh/.ssh/id_ed25519    # encrypted at rest

# 2. link a single program
tuckr add nvim

# 3. link several at once (hooks fire in order)
tuckr add zsh git nvim

# 4. inspect state — what's linked, what conflicts
tuckr status

# 5. preview before touching $HOME
tuckr add --dry-run i3

# 6. unlink cleanly (does NOT run hooks)
tuckr rm nvim

# 7. host-conditional group — only links on hosts with the
#    "laptop" tag set in ~/.config/tuckr/profile.toml
tuckr add nvim@laptop

# 8. encrypt a secret in place before committing
tuckr encrypt ssh
git -C ~/.dotfiles add . && git commit -m "rotate ssh key"
```

## Niche It Fills

**Dotfile management with batteries included.** GNU Stow is the
incumbent and does symlinks-only; everything else (yadm,
chezmoi, dotbot, rcm) trades stow's simplicity for a templating
DSL or a custom YAML config. `tuckr` keeps stow's
"one-directory-per-program, symlink to `$HOME`" contract — no
templates, no DSL — but folds in the four side-tools every stow
user ends up writing themselves: hooks, encrypted secrets,
host-conditional groups, and a real status command.

## Why use it

1. **Stow-compatible mental model.** If you can read a stow
   tree, you can read a `tuckr` tree. Migration is
   `mv ~/dotfiles ~/.dotfiles && mkdir -p Configs && mv */
   Configs/`.
2. **Hooks colocated with the program.** No `bootstrap.sh`
   god-script that grows to 400 lines and only the original
   author understands. `Hooks/nvim/post.sh` lives next to the
   nvim config, runs only when nvim is added, runs only once
   per add.
3. **Secrets in the same repo, safely.** `~/.ssh/id_ed25519`,
   `~/.aws/credentials`, GPG keys — all live in `Secrets/`,
   encrypted at rest, decrypted only at link time. The repo
   stays push-able to a public host.
4. **Conditional groups without templating.** `nvim@laptop`,
   `i3@arch` — the directory name itself is the predicate. No
   Jinja, no `{{ if .Hostname }}`.
5. **Single static binary, ~3 MB.** No Python runtime (chezmoi
   is Go but bigger, dotbot is Python). Drops onto a fresh box
   over `scp` and runs immediately.

## Vs Already Cataloged

- **Vs [`yadm`](../yadm/):** yadm wraps git — your `$HOME` *is*
  the working tree, with alt-files keyed off OS / hostname /
  user. `tuckr` keeps `$HOME` clean and symlinks in from a
  separate repo (the stow model). Use `yadm` if you want
  `$HOME`-as-repo and don't mind a thin git wrapper; use
  `tuckr` if you want explicit per-program directories and
  hooks.
- **Vs [`chezmoi`](../chezmoi/):** chezmoi is the
  templating-heavy dotfile manager (Go templates, computed
  data, encrypted-source mode, full state machine). `tuckr` is
  deliberately the *opposite* — no templates, files-on-disk
  match files-in-`$HOME` byte-for-byte, conditional inclusion
  is a directory-name suffix. Use chezmoi if you need
  per-machine *content* differences (different `git user.email`
  per host); use `tuckr` if your differences are
  *which-files-link* and you want the diff to be the diff of
  the file, not the diff of a template.
- **Vs [`stow`](../stow/):** `tuckr` is stow + hooks + secrets
  + conditional groups + status, written in Rust as a single
  binary. If you've stayed on stow only because you didn't
  want a bigger tool, this is the smallest possible
  replacement that gives you the four things you've been
  shell-scripting around.

## Caveats

- **GPL-3.0, not MIT.** Derivative works of the *tool* must
  remain open. The dotfiles you manage are unaffected.
- **Conditional groups use directory-name suffixes, not
  expressions.** `nvim@laptop` is fine; `nvim@(laptop &&
  !work)` is not. If you need boolean predicates over host
  facts, chezmoi's templating is the more powerful path.
- **Hooks run as `$USER`.** Privileged setup (`chsh`, package
  install, `launchctl load` for system agents) needs `sudo`
  inside the hook, same as any dotfile bootstrap.
- **Encrypted secrets are at-rest, not in-flight.** `tuckr add`
  decrypts to the symlink target on disk in cleartext. Use full
  disk encryption; do not rely on `Secrets/` for runtime
  protection.
