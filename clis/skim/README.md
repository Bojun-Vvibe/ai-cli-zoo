# skim

> **General-purpose interactive fuzzy finder, in Rust** — `sk` reads
> a list on stdin (filenames, git branches, processes, history
> entries, anything line-shaped) and lets you type-to-narrow with
> a live preview pane. Pinned to **v4.6.1** (2026-04-26 release,
> [LICENSE](https://github.com/skim-rs/skim/blob/master/LICENSE),
> MIT).

Source: <https://github.com/skim-rs/skim>

## TL;DR

`skim` is the Rust analogue of `fzf` — same `stdin → tty UI →
selected line on stdout` contract, same `--preview`, `--multi`,
`--bind` vocabulary, same vim / emacs / shell key-binding
package. The differences are operational: a single static Rust
binary (no Go runtime, no Ruby vim plugin), a built-in
`--interactive` mode that re-runs the upstream command on every
keystroke (great for `rg`/`grep`-driven live search), and a
library crate (`skim = "0.10"`) other Rust TUIs embed instead of
shelling out. If you already use `fzf`, the muscle memory
transfers; if you don't, this is a safe place to start.

## Install

```bash
# Homebrew (macOS / Linux)
brew install sk

# Cargo
cargo install --locked skim

# Linux package managers
# Arch:    pacman -S skim
# Debian/Ubuntu: apt install skim    # may lag the upstream tag
# Nix:     nix-env -iA nixpkgs.skim
# Alpine:  apk add skim

# from a release tarball (any OS)
curl -Lo skim.tgz "https://github.com/skim-rs/skim/releases/download/v4.6.1/skim-aarch64-apple-darwin.tgz"
tar xf skim.tgz && sudo install sk /usr/local/bin/

# verify
sk --version    # 4.6.1

# optional shell key-bindings (Ctrl-T = file picker, Ctrl-R = history,
# Alt-C = directory jumper) — clone the shell/ dir and source it:
# zsh:   source $(brew --prefix)/opt/sk/shell/key-bindings.zsh
# bash:  source $(brew --prefix)/opt/sk/shell/key-bindings.bash
```

## License

MIT — see
[LICENSE](https://github.com/skim-rs/skim/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. interactive file picker, preview the highlighted file with bat
find . -type f | sk --preview 'bat --color=always {}'

# 2. multi-select branches, delete them
git branch | sk -m | xargs git branch -D

# 3. live ripgrep — type a query, see matching lines update per keystroke
sk --ansi -i -c 'rg --color=always --line-number "{}"' \
  --preview 'bat --color=always --highlight-line $(echo {} | cut -d: -f2) $(echo {} | cut -d: -f1)'

# 4. fuzzy-pick a process to kill
ps -ef | sk -m | awk '{print $2}' | xargs kill

# 5. shell history search (with the optional key-bindings sourced)
# Ctrl-R   — fuzzy-pick a previous command, edit and re-run

# 6. embed in a script — exit code 130 = user pressed ESC
choice=$(printf 'apply\nrevert\nabort\n' | sk --prompt 'action> ') || exit 0
echo "you picked: $choice"
```

## Niche It Fills

**Generic interactive narrowing as a Unix filter.** Anything
line-shaped on stdin can become an interactive picker on
stdout in one process. The killer property is composability:
the same binary powers a file picker, a process killer, a
branch chooser, a history search, and a custom DSL for any
ad-hoc list you can produce from a shell command.

## Why use it

Three things `skim` does that pay back the install:

1. **`-i` interactive mode is `fzf` + a re-runnable upstream
   command.** Pass `-c 'rg "{}"'` and the placeholder `{}`
   re-executes `rg` on every keystroke — `skim` becomes a
   front-end for any tool that takes a query argument, not just
   a filter on a fixed list.
2. **Library crate, not just a binary.** `skim` is `cargo
   add`-able from Rust TUIs (`zellij`, `lazygit` extensions,
   internal tools) instead of shelling out and parsing back —
   one of the few popular fuzzy finders with a stable
   library API.
3. **Same key bindings as `fzf`.** Ctrl-T / Ctrl-R / Alt-C from
   `fzf`'s shell package work the same here; the install is
   risk-free if you've internalised `fzf` muscle memory.

For an LLM-CLI workflow where an agent needs to surface a
short-list to a human ("which of these five candidate files do
you want me to edit?"), piping into `sk -m` is one line and
returns the selection on stdout for the next tool call.

## Vs Already Cataloged

- **Vs [`navi`](../navi/):** orthogonal — `navi` is a
  *cheatsheet launcher* (curated commands with parameter prompts
  to fill in), `skim` is a *generic line picker*. They compose:
  `navi` uses a fuzzy finder under the hood (defaults to `fzf`
  but `--finder=sk` works) to choose which cheatsheet to run.
- **Vs [`tealdeer`](../tealdeer/):** orthogonal — `tealdeer`
  serves `tldr` example pages (one-shot lookup), `skim` is the
  picker layer you'd wrap around any list.
- **Vs `fzf` (not cataloged):** the closest peer — same UX
  contract, same shell-binding philosophy, ~10× the GitHub
  community. Pick `fzf` when ecosystem mass matters (vim
  plugins, neovim integrations, every dotfiles repo on the
  internet); pick `skim` when you want a Rust binary with no Go
  toolchain in the build, the embeddable library crate, or
  `--interactive` mode out of the box without `--bind
  'change:reload(...)'` incantations.
- **Vs [`gum`](../gum/) `filter`:** overlap — `gum filter`
  offers a similar single-pane fuzzy picker with prettier
  defaults but no `--interactive` re-execution and no
  `--preview` pipeline equivalent. `gum` wins on aesthetics for
  shell scripts; `skim` wins when you need the full
  picker-with-preview rig.

## Caveats

- **`fzf` ecosystem dwarfs `skim`'s.** Vim / Neovim / Emacs
  plugins overwhelmingly target `fzf`; `skim` ships its own
  vim plugin (`skim.vim`) but the integration depth is
  shallower. If your editor workflow already centres on
  `fzf.vim`, the switching cost is real.
- **Apple Silicon prebuilt binary lagged historically.** v4.x
  ships `aarch64-apple-darwin` tarballs; if you pin to an old
  release you may end up `cargo install`ing instead of
  Homebrew-ing.
- **`--interactive` placeholder syntax is `{}` not `fzf`'s
  `{q}`.** Easy gotcha when copy-pasting `fzf` recipes — the
  query placeholder differs.
- **Preview pane has no built-in scrollback.** Long previews
  truncate at the pane height; pipe into `bat --paging=always`
  or `less` from a `--bind 'enter:execute(...)'` hook for a
  scroll-and-return workflow.
- **Default ranking algorithm differs from `fzf`.** Same family
  (subsequence + bonus for word boundaries / camelCase) but the
  exact ranking on tied queries can differ — power users
  switching from `fzf` may need a few days to recalibrate.
