# onefetch

> **`neofetch` for git repositories** — point it at a checkout
> and it prints the project's primary-language ASCII logo next to
> a dense column of repo stats: top contributors, line counts per
> language, churn, last change, license, pending changes, size on
> disk. Pinned to **v2.27.1** (commit
> `497d4c011ede6acba4b3ca4c1e62f9aecaf20528`,
> [LICENSE.md](https://github.com/o2sh/onefetch/blob/main/LICENSE.md),
> MIT).

Source: <https://github.com/o2sh/onefetch>

## TL;DR

`onefetch` is the "cd into a repo I have never seen before, what
even is this thing" tool. One synchronous shell-out, no network,
no API tokens — it walks `.git/` directly with `gix` and counts
files with `tokei`, so it works on private mirrors and air-gapped
boxes. The output is decorative *and* informative: ASCII logo on
the left, then `Project / HEAD / Pending / Version / Created /
Language / Authors / Last change / Repo / Commits / LOC / Size /
License`. v2.27.1 (the pin here) adds Roc, Mojo, and several new
language logos plus a `--show-logo never` mode for CI summaries.
Useful in a shell prompt hook, in a `tmux` new-window template,
or as a screenshot for a project's README header.

## Install

```bash
# Homebrew (macOS / Linux)
brew install onefetch

# Cargo (any platform with a Rust toolchain)
cargo install --locked onefetch     # picks up 2.27.1 from crates.io

# Linux distro packages
# Arch:    pacman -S onefetch
# Nix:     nix-env -iA nixpkgs.onefetch
# Alpine:  apk add onefetch

# Static binary from the release page
curl -L -o onefetch \
  https://github.com/o2sh/onefetch/releases/download/2.27.1/onefetch-linux.tar.gz \
  && tar xf onefetch && sudo install onefetch /usr/local/bin/

# verify
onefetch --version    # onefetch 2.27.1
```

## Representative examples

```bash
# 1. Default: run inside any git checkout
cd ~/Projects/some-repo && onefetch

# 2. Hide the logo, only print the stats — handy for piping into
#    a chat / CI summary
onefetch --show-logo never --no-color-palette

# 3. JSON output for scripting (e.g. a dashboard that surfaces
#    the most-touched language across many repos)
onefetch --output json | jq '{repo: .project.name,
                              top_lang: .languages[0].language,
                              loc: .lines_of_code}'
```

## When to use vs. alternatives

- Pick **onefetch** when you want a *single* glance at "what is
  this repo about" — language mix, who wrote it, how active it is,
  in one screen.
- Pick [`tokei`](../tokei/) when you only need lines-of-code per
  language, fast, with no logo and no git walk — it's `onefetch`'s
  upstream counter.
- Pick [`scc`](https://github.com/boyter/scc) (not yet in this
  catalog) when you also want complexity estimates and COCOMO
  cost projections alongside LOC.
- Pick `git shortlog -sn --all` for *just* the contributor leader-
  board, when you don't want any of the rest.
- Pick a hosted service (GitHub Insights, Sourcegraph) when the
  question is cross-repo or historical — `onefetch` is one repo,
  one moment, one screen.
