# mas

> **Mac App Store command-line interface** — a single Swift
> binary that lets you search, install, upgrade, and account-switch
> Mac App Store apps from the terminal, the same way `brew` handles
> Homebrew packages. Pinned to **v7.0.0** (released 2026-05-04,
> [LICENSE](https://github.com/mas-cli/mas/blob/main/LICENSE),
> MIT).

Source: <https://github.com/mas-cli/mas>

## TL;DR

`mas` is the missing apt/brew for the Mac App Store. macOS ships
the App Store as a GUI-only application, so reproducible machine
setup (Brewfile, Ansible, dotfiles bootstrap, fresh-laptop
scripts) historically had to skip everything that only ships
through the store — Xcode, Keynote, Pages, Numbers, Things 3,
Pixelmator Pro, Magnet, Affinity apps, etc. `mas` plugs that
hole: it talks to the same StoreKit framework the GUI uses, so
purchases, updates, and Apple ID sign-in state are shared. With
`brew bundle` integration it lets you encode "this laptop" as
one `Brewfile` checked into git.

## Install

```bash
# Homebrew (the canonical install)
brew install mas

# verify
mas version    # 7.0.0
mas account    # show signed-in Apple ID
```

## Examples

```bash
# search, then install by app id (the store id, not the name)
mas search "Things 3"
mas install 904280696

# list installed Mac App Store apps with their ids + versions
mas list

# show what the store has updates for, then apply them all
mas outdated
mas upgrade

# upgrade only one app
mas upgrade 497799835    # Xcode

# encode the lot into a Brewfile so a fresh Mac is one command
brew bundle dump --file=~/Brewfile     # writes `mas "Xcode", id: 497799835`
brew bundle --file=~/Brewfile          # restore on a new machine
```

## Use when

- You are scripting fresh-Mac setup and refuse to leave Mac App
  Store apps as a manual click-through step at the end. Pair
  `mas` with `brew bundle` and your dotfiles repo to get
  one-command laptop provisioning.
- You want CI / MDM / Ansible to install or update Xcode and the
  iWork suite without a human at the keyboard.
- You manage a small fleet (consultancy, design studio, indie dev
  shop) and want to roll out App Store updates over SSH instead
  of asking each user to open the store.

Skip `mas` when the app you need is *not* in the Mac App Store
(use Homebrew Cask instead — `brew install --cask zoom`), or when
you need to install something the signed-in Apple ID has never
purchased: since macOS Mojave, `mas` cannot perform a first-time
"buy" of a free or paid app for an Apple ID that has not already
acquired it once through the GUI. Real-world flow is "click
'Get' once on a reference machine, then `mas install <id>`
everywhere else under the same Apple ID forever after."
