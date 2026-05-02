# nh

> **"Yet Another Nix Helper" — a Rust CLI that wraps
> `nixos-rebuild`, `home-manager`, and `darwin-rebuild` with
> a unified flake-aware UX, live `nvd` diff of the upcoming
> generation, and a built-in flake-input search/update
> surface.** Pinned to **v4.3.2**
> ([LICENSE](https://github.com/nix-community/nh/blob/master/LICENSE),
> EUPL-1.2).

Source: <https://github.com/nix-community/nh>

## TL;DR

`nh` is the daily-driver wrapper Nix users reach for once
they stop typing `sudo nixos-rebuild switch --flake
.#$(hostname)` ten times a day. One binary, three
subcommands that mirror the three rebuild surfaces — `nh os`
(NixOS), `nh home` (Home-Manager), `nh darwin` (nix-darwin)
— each with `switch` / `boot` / `test` / `build` and a
default of "find the flake in the current directory or
`~/.config/nix-config`, build it, show me a `nvd` diff of
what changed, then ask before activating". The diff step
alone is the killer feature: every rebuild prints "you are
about to add 47 packages, remove 3, upgrade `linux 6.11.0 →
6.12.1`" *before* the activation runs, which is the loop
plain `nixos-rebuild` never had. Bonus: `nh search` is a
fast TUI front-end to the official `search.nixos.org`
backend (no more keeping a browser tab open), and `nh clean`
wraps `nix-collect-garbage -d` with the right flags for the
right user / root / boot-loader split.

## Install

```bash
# NixOS / nix-darwin / Home-Manager (the canonical path —
# nh is in nixpkgs)
nix profile install nixpkgs#nh

# Or in your flake's modules:
#   programs.nh.enable = true;          # Home-Manager
#   programs.nh = { enable = true; };   # NixOS / nix-darwin

# Static binary from releases (any Linux / macOS, no nixpkgs needed)
curl -L -o /usr/local/bin/nh \
  https://github.com/nix-community/nh/releases/download/v4.3.2/nh-$(uname -m)-$(uname -s | tr '[:upper:]' '[:lower:]')
chmod +x /usr/local/bin/nh

# Verify
nh --version    # nh 4.3.2
```

## Example usage

```bash
# 1. Rebuild the current host's NixOS config from the flake
#    in $FLAKE (or ~/.config/nix-config), show diff, then switch
export FLAKE=~/nix-config
nh os switch

# 2. Same on macOS
nh darwin switch

# 3. Home-Manager only (no root needed)
nh home switch

# 4. Build but do not activate — useful in CI / pre-merge checks
nh os build .#myhost

# 5. Search nixpkgs interactively (TUI)
nh search ripgrep

# 6. GC: drop everything older than 7 days, plus old boot entries
nh clean all --keep-since 7d --keep 5
```

## Why this lives in the zoo

Vanilla `nixos-rebuild` / `home-manager switch` /
`darwin-rebuild` are three different commands with three
different flag sets and zero feedback about *what* the new
generation actually changes — you find out by reading the
journal after activation. `nh` collapses them into one
muscle-memory surface, adds the pre-activation diff that
makes "is this rebuild safe to switch into right now?"
answerable in two seconds, and folds in the search + GC
chores the upstream UX makes you context-switch for. For
anyone running NixOS / nix-darwin / Home-Manager on a daily
basis, it is the single quality-of-life upgrade that pays
back every rebuild for the rest of the year.
