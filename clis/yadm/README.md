# yadm

> **Yet Another Dotfiles Manager** — a thin Bash wrapper around
> `git` that tracks your `$HOME` config files in a bare repository
> at `~/.local/share/yadm/repo.git` (work-tree = `$HOME`) so you
> never need a symlink farm or a separate "dotfiles" subtree.
> Pinned to **v3.5.0** (released 2025-03-03,
> [LICENSE](https://github.com/yadm-dev/yadm/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/yadm-dev/yadm>

## TL;DR

`yadm` solves three things that every "store dotfiles in git"
recipe eventually trips on:

1. **No symlinks, no `$HOME`-as-repo.** It uses git's bare-repo +
   alternate work-tree pattern, so `git status` in `$HOME` keeps
   working for any *other* repo you might have there, while
   `yadm status` only ever sees the files you've explicitly
   `yadm add`-ed.
2. **Per-host / per-OS variants.** `~/.gitconfig##os.Darwin` and
   `~/.gitconfig##class.work` are picked automatically based on
   detected OS, distro, hostname, user, or arbitrary class tags
   you set with `yadm config local.class`. No more `if uname …`
   inside every dotfile.
3. **Encrypted secrets in the same repo.** `yadm encrypt` runs
   the patterns in `~/.config/yadm/encrypt` through GPG (or age
   via a hook) into a single `~/.local/share/yadm/archive`, so
   ssh keys / API tokens travel with the rest of your dotfiles
   without ever sitting plaintext in git history.

Plus alt-file *templates* (`##template.default`, `##template.esh`,
`##template.j2`) for files that need light variable substitution
per host, and a `bootstrap` hook that runs once after `yadm clone`
so a fresh laptop is one command away from your full setup.

## How it compares vs alternatives in this zoo

- vs [`chezmoi`](../chezmoi/) — chezmoi is a Go binary with its
  own DSL, source state directory (`~/.local/share/chezmoi/`), and
  a richer template engine (Go templates, `chezmoi data`, password
  manager integration). `yadm` is ~3 kLOC of Bash that delegates
  to `git` for *everything* — if you can already drive git from
  the CLI, you already know 90% of yadm. Pick chezmoi when you
  want declarative state and managed templates; pick yadm when
  you want "git, but for `$HOME`" with the smallest possible
  surface area.
- vs hand-rolled `git --git-dir=$HOME/.dotfiles --work-tree=$HOME`
  aliases — yadm gives you the same model plus alt files,
  encryption, bootstrap, and Bash/Zsh/Fish completion for free,
  and it ships in Homebrew / apt / dnf / pacman / nix so a fresh
  box is `brew install yadm && yadm clone …`.
- vs [`sops`](../sops/) / [`age`](../age/) for secrets — those
  are *encryption tools* (sops for structured KV files with KMS /
  age recipients; age for raw file encryption). yadm `encrypt`
  is an *archive* of arbitrary `$HOME` paths (ssh keys, `.netrc`,
  GPG private keys) that rides along with your dotfiles repo via
  GPG by default — no KMS, no recipients file, just a passphrase.
  Use sops/age when you want per-file rotation and team access;
  use yadm-encrypt for the boot-strappy stuff one machine needs
  once.

## Install

```bash
# Homebrew (macOS / Linux)
brew install yadm

# Linux package managers
# Debian / Ubuntu: apt install yadm
# Fedora: dnf install yadm
# Arch (AUR): yay -S yadm
# Nix: nix-env -iA nixpkgs.yadm

# Manual single-file install (no deps beyond bash + git)
sudo curl -fLo /usr/local/bin/yadm \
  https://github.com/yadm-dev/yadm/raw/master/yadm
sudo chmod a+x /usr/local/bin/yadm

# verify
yadm version    # yadm 3.5.0
```

## Examples

```bash
# start tracking dotfiles on a new machine
yadm init
yadm add ~/.zshrc ~/.gitconfig ~/.config/nvim
yadm commit -m "initial dotfiles"
yadm remote add origin git@github.com:you/dotfiles.git
yadm push -u origin main

# clone onto a fresh laptop and run the bootstrap hook
yadm clone git@github.com:you/dotfiles.git
# yadm sees ~/.config/yadm/bootstrap and offers to run it

# per-OS variant: macOS gets a different .gitconfig
cp ~/.gitconfig ~/.gitconfig##os.Darwin
yadm add ~/.gitconfig##os.Darwin
yadm alt   # regenerates the active ~/.gitconfig from the matching variant

# encrypt ssh keys into the repo
echo '.ssh/id_*' >> ~/.config/yadm/encrypt
yadm encrypt   # produces ~/.local/share/yadm/archive (commit it)

# day-to-day: identical to git
yadm status
yadm diff ~/.zshrc
yadm commit -am "tweak prompt"
yadm push
```

## When NOT to reach for yadm

- You want a single binary with no Bash dependency — use
  [`chezmoi`](../chezmoi/).
- You're managing *system* config (`/etc/…`) — use a real config
  management tool (Ansible, NixOS, etcetera). yadm is `$HOME`-only
  by design.
- You don't already use git from the CLI — then yadm's "it's just
  git" pitch is more friction than feature.
