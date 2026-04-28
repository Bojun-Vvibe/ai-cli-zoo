# chezmoi

- **Repo:** https://github.com/twpayne/chezmoi
- **Version:** v2.70.2 (latest stable, 2026)
- **License:** MIT ([LICENSE](https://github.com/twpayne/chezmoi/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install chezmoi` · `sh -c "$(curl -fsLS get.chezmoi.io)"` · `pacman -S chezmoi` · `go install github.com/twpayne/chezmoi/v2@latest` · prebuilt binaries on the GitHub release page · binary name is `chezmoi`

## What it does

`chezmoi` manages your dotfiles across multiple machines from a
single source-of-truth git repo. The model is one-way: your repo
holds *templates* of config files (`.zshrc.tmpl`,
`.gitconfig.tmpl`, etc.), and `chezmoi apply` renders them into
`$HOME` using machine-local data (hostname, OS, secrets fetched
from `pass` / `1password` / `bitwarden` / `keepassxc` / GPG-encrypted
files). The same repo can produce a slightly different `.gitconfig`
on your work mac, your home linux box, and an ephemeral devcontainer
without per-machine branches. Diffs flow both ways: `chezmoi diff`
shows what would change in `$HOME`, `chezmoi add ~/.foo` pulls a
modified file back into the source state.

## When to pick it / when not to

Reach for `chezmoi` when you have ≥2 machines you want to keep
in lockstep, when some files must differ per-host (different
`$EDITOR`, different `git user.email` for work vs personal), and
when at least one config file contains a secret you do not want to
commit in plaintext. It is also the right tool when you regularly
spin up fresh devcontainers / cloud-shell / VMs and want a one-line
bootstrap (`sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply <user>`)
that lands you in a fully-configured shell.

Skip it for a single laptop where a plain git repo + symlinks works
fine. Skip it if your secret store cannot be invoked from a CLI
(`chezmoi`'s template functions need a callable `pass`/`op`/`bw`
binary). Skip it on systems where you cannot install Go binaries.
For pure symlink farms, [`stow`](https://www.gnu.org/software/stow/)
is simpler; for a Nix-managed system, `home-manager` subsumes this.

## Why it matters in an AI-native workflow

Coding agents that spawn ephemeral sandboxes — `container-use`
microVMs, E2B Firecracker VMs, GitHub Codespaces, Claude Code
devcontainers — start each session with a bare `$HOME`. Without
something like `chezmoi`, every fresh sandbox loses your
`.gitconfig`, shell aliases, `claude.json` model preferences,
`opencode` plugin config, and the `~/.config/<tool>` dirs that
your coding CLIs read on startup. The agent then either runs
without your preferences (worse output, wrong model, no aliases)
or wastes turns recreating them. A single `chezmoi init --apply
github.com/<you>/dotfiles` line in the sandbox bootstrap restores
the entire host-shaped environment in one step, including
per-sandbox-host overrides via `.chezmoi.toml` templates.

## Example invocations

```bash
# First-time setup on a new machine: clone your dotfiles repo and apply
chezmoi init --apply github.com/<you>/dotfiles

# See what would change in $HOME without writing
chezmoi diff

# Apply the source state to $HOME
chezmoi apply

# Pull a locally-edited dotfile back into the source repo
chezmoi add ~/.zshrc

# Edit a managed file (opens $EDITOR on the source template, not the target)
chezmoi edit ~/.gitconfig

# Pull upstream changes and re-apply in one step
chezmoi update

# Render a single template to stdout for debugging
chezmoi cat ~/.gitconfig

# Show all data the templates can reference (OS, hostname, $USER, custom)
chezmoi data
```

## Alternatives in this catalog

- [`mise`](../mise/) — manages tool *versions* per-project; pairs
  with `chezmoi` (chezmoi for `~/.config/mise/config.toml`, mise
  for `node@22 && python@3.12`).
- [`direnv`](../direnv/) — per-directory env vars; complements
  `chezmoi` (chezmoi for the dotfile that loads direnv, direnv
  for the project-local `.envrc`).
- [`atuin`](../atuin/) — syncs shell *history* across machines;
  the natural pair to chezmoi (chezmoi syncs config, atuin syncs
  what you actually typed).
