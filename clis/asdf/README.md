# asdf

- **Repo:** https://github.com/asdf-vm/asdf
- **Version:** v0.19.0 (latest stable, 2026-04-24)
- **License:** Apache-2.0 ([LICENSE](https://github.com/asdf-vm/asdf/blob/master/LICENSE))
- **Language:** Go (rewritten from Bash in v0.16, 2025)
- **Install:** `brew install asdf` · `pacman -S asdf-vm` · static binary releases on the GitHub release page (`asdf-v0.19.0-darwin-arm64.tar.gz` etc.) · `apt install asdf` (Ubuntu 24.10+) · build from source with `go build`

## What it does

`asdf` is a single-binary multi-runtime version manager. Instead of
installing one of `nvm`, `rbenv`, `pyenv`, `tfenv`, `tgenv`, `goenv`,
`jenv`, `kubectl-version`, `helm-version`, `swiftenv`, and so on, you
install `asdf` once and then add a *plugin* per language or tool: `asdf
plugin add nodejs`, `asdf plugin add python`, `asdf plugin add
terraform`, `asdf plugin add kubectl`. Each plugin is a small shell
repo on GitHub that knows how to list, download, and compile / extract
that runtime — the plugin index lists ~600 plugins covering every
mainstream language plus most CLI tools that ship versioned binaries
(terraform, helm, kubectl, kustomize, yq, jq, sops, age, golangci-lint,
flyctl, doctl, gcloud, awscli, packer, vault, consul, nomad, …).
Versions are pinned per project via a `.tool-versions` file at the
repo root: a plain text file like `nodejs 22.21.1\npython 3.13.7\nterraform
1.13.4`, which `asdf` walks up from `cwd` to find. The shim layer in
`~/.asdf/shims/<binary>` reads that file at exec time and dispatches
to the right installed version, so every shell tool on PATH respects
the per-project pin without any `nvm use` step. Global defaults live
in `~/.tool-versions`. v0.19.0 is part of the Go-rewrite line (the
project moved off Bash in v0.16, January 2025) so install is now a
single static binary instead of a sourced shell script — the trade-off
is that the v0.16+ binary no longer auto-loads on shell startup; you
either prepend `~/.asdf/shims` to PATH yourself or use the legacy
shell-extension package.

## When to pick it / when not to

Pick `asdf` when a single repo or laptop needs many runtimes pinned to
specific versions and you want one tool to manage all of them, with one
config file format (`.tool-versions`) the whole team commits to git.
Especially compelling on polyglot monorepos where the alternative is a
README that says "install nvm, install rbenv, install pyenv, install
tfenv…" and a new joiner spends a day getting the right combination
working. The plugin model means support for new tools usually lands
within a week of their first release — `asdf plugin add <newthing>` is
how you get version pinning for tools whose authors did not ship one.
Pair with [`mise`](../mise/) only as comparison (see below); pair with
[`direnv`](../direnv/) to layer per-project env vars on top of the
per-project tool versions; pair with [`pkgx`](../pkgx/) when you want
ephemeral on-demand tool execution rather than persistent installs;
pair with [`pixi`](../pixi/) when the runtime in question is a Conda
package (Python scientific stack, CUDA toolkits) where Conda's solver
beats per-language version managers.

Skip it for `mise` users — [`mise`](../mise/) is a fork-flavoured
rewrite (originally `rtx`) that is `.tool-versions`-compatible, faster
on shell startup, has a richer task-runner and env-var layer built in,
and is now the default recommendation for new teams; `asdf` is the
boring stable option you already have at $WORK, `mise` is the one
tutorials in 2026 default to. Skip it for single-language shops — if
99% of your devboxes only need one Python or one Node, the language's
own version manager (`pyenv`, `nvm`, `volta`) is one less abstraction.
Skip the v0.16+ Go binary if your existing setup depends on
`asdf reshim` or the shell-function `asdf` invocation; the Go rewrite
changed several edge cases and some plugins still assume the Bash
host. Skip it inside containers — for reproducible CI / image builds,
just pin the runtime in the base image; `asdf` shines on the
many-projects-per-laptop axis, not the one-project-per-container axis.

## Example invocations

```bash
# One-time bootstrap: add the asdf shims to PATH (zsh shown)
echo 'export PATH="$HOME/.asdf/shims:$PATH"' >> ~/.zshrc

# Add plugins for the runtimes this laptop needs
asdf plugin add nodejs
asdf plugin add python
asdf plugin add terraform
asdf plugin add kubectl

# List available versions for a plugin
asdf list all nodejs | tail -20
asdf list all terraform | grep '^1\.'

# Install a specific version (downloads + builds in ~/.asdf/installs)
asdf install nodejs 22.21.1
asdf install python 3.13.7
asdf install terraform 1.13.4

# Pin per-project (writes .tool-versions in cwd)
cd ~/code/myrepo
asdf set nodejs 22.21.1
asdf set python 3.13.7
asdf set terraform 1.13.4
cat .tool-versions
# nodejs 22.21.1
# python 3.13.7
# terraform 1.13.4

# Set a global default (writes ~/.tool-versions)
asdf set --home nodejs 22.21.1

# Inspect what asdf would dispatch for a binary in cwd
asdf which node
asdf current

# Update a plugin's version index (run before listing brand-new releases)
asdf plugin update nodejs

# Install everything mentioned in the cwd .tool-versions in one shot
asdf install
```
