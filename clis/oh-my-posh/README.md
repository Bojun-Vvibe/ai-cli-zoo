# oh-my-posh

> **Cross-shell prompt theme engine.** A single static binary
> that renders a fully customizable prompt for bash, zsh, fish,
> pwsh, nu, elvish, xonsh, and cmd, driven by a JSON / YAML /
> TOML config. Ships 80+ built-in segments (git status, k8s
> context, AWS profile, language version detectors for ~30
> runtimes, battery, weather, exec time, exit code, etc.) and
> 100+ themes. Pinned to **v25.16.0**
> ([COPYING](https://github.com/JanDeDobbeleer/oh-my-posh/blob/main/COPYING),
> MIT).

Source: <https://github.com/JanDeDobbeleer/oh-my-posh>

## TL;DR

One binary, one config, identical prompt across every shell on
every machine — the alternative to maintaining parallel
`.zshrc` / `.bashrc` / `pwsh` prompt code. Segments are
declarative blocks in the theme file; each one can have its own
foreground/background, powerline separator, template, and
visibility condition. Unlike pure-shell prompts (`pure`,
`spaceship`), it does not slow zsh startup with hundreds of
function defs — it is one process call per render with built-in
caching for expensive segments (git, kube).

## Install

```bash
# Homebrew (macOS / Linux)
brew install oh-my-posh

# Install script (Linux/macOS)
curl -s https://ohmyposh.dev/install.sh | bash -s

# Winget (Windows)
winget install JanDeDobbeleer.OhMyPosh -s winget

# Pre-built binary
curl -L https://github.com/JanDeDobbeleer/oh-my-posh/releases/download/v25.16.0/posh-linux-amd64 -o /usr/local/bin/oh-my-posh
chmod +x /usr/local/bin/oh-my-posh
```

## Example

```bash
# zsh — pick a built-in theme
eval "$(oh-my-posh init zsh --config $(brew --prefix oh-my-posh)/themes/jandedobbeleer.omp.json)"

# bash
eval "$(oh-my-posh init bash --config ~/my-theme.omp.json)"

# Pwsh — add to $PROFILE
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH/atomic.omp.json" | Invoke-Expression

# List bundled themes
oh-my-posh get themes
```

## When to use

- You hop between zsh on macOS, bash on a Linux server, and
  pwsh on Windows and want the *same* prompt everywhere.
- You want rich segments (git ahead/behind, current k8s
  namespace, active AWS profile) without writing them in shell.
- Your zsh prompt is noticeably slow because of plugin-driven
  segment code; oh-my-posh's caching + single-binary render is
  typically faster than equivalent pure-shell setups.

## When NOT to use

- You want a *minimal* prompt with no Nerd Font requirement —
  `pure` or a hand-written PS1 stays out of your way.
- You prefer your prompt to be plain shell so it works on any
  random server with no extra binary — keep it in `.bashrc`.
- You already use `starship` and like the TOML model; the
  feature overlap is large enough that switching is rarely
  worth the config churn.
