# toml-bombadil

> **toml-bombadil** — oknozor/toml-bombadil, a dotfile manager built
> around **TOML templating + per-host profiles + symlinked output**:
> source dotfiles live in one repo with `{{ var }}` Tera-template
> placeholders, a `bombadil.toml` declares the variable values per
> profile (laptop / workstation / server), and `bombadil link`
> renders templates to a staging tree and symlinks them into
> `$HOME` — so the same `alacritty.yml` source produces a
> dark-theme + 14pt rendering on the laptop and a light-theme +
> 11pt rendering on the workstation, both reproducible from one
> git push. Pinned to **4.2.0**, MIT — license file:
> [LICENSE](https://github.com/oknozor/toml-bombadil/blob/main/LICENSE).

Source: <https://github.com/oknozor/toml-bombadil>

## TL;DR

The dotfile-manager design space splits on three questions: **how
are per-machine differences expressed?**, **how do files reach
$HOME?**, and **what runs after a file is placed?**. toml-bombadil's
answer is:

1. **Per-machine differences are profile-scoped variables** —
   `bombadil.toml` declares a default `[settings.dotfiles]` and a
   `[profiles.<name>]` block per machine adding or overriding
   variables. `bombadil link --profiles work` activates a profile.
2. **Files reach $HOME via render-then-symlink** — every source
   file goes through Tera (Jinja-shaped) templating into
   `$XDG_DATA_HOME/bombadil/dotfiles/`, and that staging copy is
   symlinked to its final path. The source of truth is the rendered
   file (which `git diff` can read) but the active file is a
   symlink (which lets you swap profiles atomically).
3. **Post-link hooks run shell commands** — declared per-profile in
   `[settings.dotfiles.<name>.hooks]`, so applying the `nvim`
   dotfile triggers `nvim --headless +PlugInstall +qall` and
   applying the `gnupg` dotfile triggers `gpgconf --reload`.

A bonus: **GPG-encrypted secrets** via `gpg --encrypt` integrated
into the source tree — secret values live encrypted in the public
dotfiles repo, decrypt at link time using your local GPG key, and
the rendered file in the staging tree is plaintext but never
committed.

## Install

```bash
# Single static Rust binary — releases at
# https://github.com/oknozor/toml-bombadil/releases/tag/4.2.0

# Cargo
cargo install toml-bombadil --version 4.2.0

# Homebrew tap
brew install oknozor/toml-bombadil/toml-bombadil

# Arch (AUR)
paru -S toml-bombadil-bin
```

## Example workflow

```bash
# In a fresh dotfiles repo
bombadil install -p ~/Projects/dotfiles

# Define a profile in ~/Projects/dotfiles/bombadil.toml
# [settings]
# dotfiles_dir = "."
#
# [settings.vars]
# theme = "gruvbox-dark"
# font_size = 14
#
# [profiles.work.vars]
# theme = "solarized-light"
# font_size = 11
#
# [settings.dotfiles.alacritty]
# source = "alacritty.yml.tmpl"
# target = ".config/alacritty/alacritty.yml"

# Render + symlink with the default profile
bombadil link

# Switch to the work profile (re-renders + re-symlinks)
bombadil link --profiles work

# Add a new dotfile to be managed
bombadil add ~/.config/git/config

# Inspect what got rendered for the active profile
bombadil get vars

# Roll back: bombadil remembers the symlink targets it created
bombadil unlink
```

## Niche

Dotfile manager — TOML-driven templating with per-host profiles,
render-then-symlink output, and integrated GPG secret support.

## Why it matters

Every dotfile manager occupies one corner of the
**templating-vs-symlinking** quadrant:

- [`stow`](../stow/) — pure symlinking, no templating; per-host
  differences need separate directories or wrapper scripts
- [`tuckr`](../tuckr/) — symlinks + per-program hooks + encrypted
  secrets + directory-suffix conditional groups; templating is
  not the primary verb
- [`yadm`](../yadm/) — `$HOME`-as-git-repo with a Jinja
  alt-files mechanism for per-host content
- [`chezmoi`](../chezmoi/) — render-templates-to-$HOME (no
  symlinks); rich Go template language with secret-manager
  integrations
- **toml-bombadil** — render-then-symlink with TOML-declared
  profile variables and Tera templates

Pick toml-bombadil when (a) you want **profile-shaped per-host
variation** (named profiles activated by flag, not implicit per-host
detection), (b) you want **symlinks not copies** so editing the
rendered file is editing the staging tree (and `git status` in the
dotfiles repo flags drift), and (c) the templating language is
secondary to the **declarative TOML inventory** of which file goes
where with which variables.

Pick chezmoi over toml-bombadil when the templating engine matters
more than the symlink semantics (chezmoi's Go templates have
first-party 1Password / Bitwarden / pass / Vault integrations that
bombadil's GPG-only secret path doesn't match). Pick stow / tuckr
when the per-machine differences are *which files link*, not
*content of files*. Pick yadm when the simpler "git repo at $HOME"
mental model wins.

Pair with [`gopass`](../gopass/) (alternative to bombadil's GPG
secret integration — keep secrets in a separate password store and
reference them from hooks at link time), [`mise`](../mise/) (manage
the runtime versions the dotfiles assume — Node, Python, etc.),
[`atuin`](../atuin/) (sync shell history across the same set of
machines), and [`hk`](../hk/) (run hooks at git commit time over
the dotfiles repo to validate templates render).

Caveats — render-then-symlink means a hand-edit to the symlinked
file in `$HOME` does not propagate back to the source template
(edit the template and re-run `bombadil link`); Tera template
errors only surface at link time so a missing variable bricks the
target file (`bombadil get vars` to dry-check); GPG secret
integration assumes a working `gpg-agent` (paranoid setups with
hardware tokens work but the touch-confirmation interrupts every
`bombadil link`); single-maintainer cadence so pin the version in
your bootstrap script.

## Verified facts

- Repo: <https://github.com/oknozor/toml-bombadil>
- Latest release tag: `4.2.0`
- License: MIT — `LICENSE` at repo root
- Language: Rust
