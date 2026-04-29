# vivid

> **A themeable `LS_COLORS` / `LSCOLORS` generator** — picks a YAML
> theme (`molokai`, `solarized-dark`, `snazzy`, `ayu`, `iceberg`, …)
> and a database of file-type → colour mappings, then prints the
> dircolors-shaped escape string `ls` / `eza` / `lsd` consume. Pinned
> to **v0.11.1**
> ([LICENSE-MIT](https://github.com/sharkdp/vivid/blob/master/LICENSE-MIT),
> dual MIT / Apache-2.0).

Source: <https://github.com/sharkdp/vivid>

## TL;DR

`LS_COLORS` is the environment variable `ls --color=auto`, `tree`,
`fd`, `eza`, `lsd`, `zsh`'s completion, and most other directory
listers read to colour filenames by extension and file type.
Hand-editing it produces a 4 KB single-line string nobody can read
or version. `vivid` separates the concern in two: a **filetype
database** (`~/.config/vivid/filetypes.yml` — `*.rs` is "source
code", `*.tar.gz` is "archive", `Dockerfile` is "build config") and
a **theme** (`~/.config/vivid/themes/<name>.yml` — "source code is
foreground colour `#a6e22e`, archive is bold red, build config is
italic cyan"). One command — `vivid generate <theme>` — emits the
`LS_COLORS` blob; one shell-init line —
`export LS_COLORS="$(vivid generate molokai)"` — makes it stick.
Switch themes by changing the argument; the filetype taxonomy stays.

## Install

```bash
# Homebrew (macOS / Linux)
brew install vivid

# Cargo
cargo install --locked vivid

# Linux package managers
# Arch: pacman -S vivid
# Nix: nix-env -iA nixpkgs.vivid

# verify
vivid --version    # vivid 0.11.1
vivid themes       # list bundled themes
```

Wire it into the shell:

```bash
# bash / zsh
echo 'export LS_COLORS="$(vivid generate molokai)"' >> ~/.zshrc

# fish
set -Ux LS_COLORS (vivid generate molokai)
```

`eza` and `lsd` honour `LS_COLORS` by default; `ls --color=auto`
honours it on Linux (GNU coreutils) and on macOS when you install
`coreutils` and use `gls`.

## License

Dual MIT / Apache-2.0 — see
[LICENSE-MIT](https://github.com/sharkdp/vivid/blob/master/LICENSE-MIT)
and
[LICENSE-APACHE](https://github.com/sharkdp/vivid/blob/master/LICENSE-APACHE).
Permissive, redistributable, no attribution required for binaries.

## One Concrete Example

```bash
# 1. preview every bundled theme against the same file list
for t in $(vivid themes); do
  printf '\n=== %s ===\n' "$t"
  LS_COLORS="$(vivid generate "$t")" ls --color=always
done

# 2. pin a theme into the shell rc
echo 'export LS_COLORS="$(vivid generate snazzy)"' >> ~/.zshrc

# 3. theme switch by env var, on the fly
LS_COLORS="$(vivid generate ayu)" eza --long --git

# 4. dump the active filetype database to a fresh YAML you can edit
vivid -d ~/myfiletypes.yml -m 8-bit generate molokai > /dev/null
mkdir -p ~/.config/vivid
cp "$(vivid --print-database)" ~/.config/vivid/filetypes.yml
$EDITOR ~/.config/vivid/filetypes.yml      # add *.tfvars, *.kcl, *.hcl, etc.

# 5. dump and edit a theme
vivid --print-theme molokai > ~/.config/vivid/themes/molokai-dim.yml
$EDITOR ~/.config/vivid/themes/molokai-dim.yml
LS_COLORS="$(vivid generate molokai-dim)" ls --color=always

# 6. constrain colour depth (useful inside tmux on a 256-colour TERM)
LS_COLORS="$(vivid -m 8-bit generate molokai)" ls --color=always
```

## Niche It Fills

**The split-config layer for `LS_COLORS`.** Without `vivid`, you
either (a) inherit a distro default that nobody can explain, (b)
copy-paste a 4 KB blob from a dotfiles repo and never touch it
again, or (c) hand-edit `dircolors` syntax. With `vivid`, you edit
two readable YAMLs: "what file types exist" and "what colours each
gets". Theme switching becomes `vivid generate <other>` and the
taxonomy you curated for `*.tfvars` / `*.k8s.yaml` / `*.parquet` is
reused across every theme.

## Why use it

1. **Filetype taxonomy is reusable across themes.** Adding `*.parquet`
   = "data file" once means every theme you generate next gets the
   correct colour for parquet files; no per-theme search-and-replace.
2. **Themes are diff-friendly YAML.** A theme is ~150 lines of
   readable category → hex assignments instead of a one-line
   colon-delimited dircolors blob; `git diff` between themes is
   meaningful.
3. **Bundled themes cover the common ground.** `molokai`,
   `solarized-dark`, `solarized-light`, `snazzy`, `ayu`, `iceberg`,
   `one-dark`, `lava`, `gruvbox-dark`, `jellybeans`, `nord`,
   `dracula` — pick one and you have a coherent palette without
   curating individual file-type colours.

For an LLM-CLI workflow, `vivid generate <theme>` is the one-liner
that makes [`eza`](../eza/) / [`bat`](../bat/) screenshots in your
issue templates and PR descriptions reproducibly themed across
machines — pin the theme in `~/.zshrc` and the dotfiles repo carries
the choice.

## Vs Already Cataloged

- **Vs [`bat`](../bat/) / [`eza`](../eza/):** orthogonal — those are
  *consumers* of `LS_COLORS` (and bat additionally has its own theme
  for syntax highlighting); `vivid` is the *producer*. They compose:
  `vivid generate molokai` themes the file list, `bat --theme=Monokai`
  themes the file contents.
- **Vs `dircolors` (GNU coreutils):** `dircolors` reads a `.dircolors`
  file with one-line per pattern and emits the same `LS_COLORS`
  string. `vivid` is the modern replacement: split filetype database
  from theme so themes are interchangeable, ship a useful default
  taxonomy, ship 12 themes.

## Caveats

- **No `LS_COLORS` consumer ships in macOS by default.** macOS `ls`
  uses the legacy `LSCOLORS` (note no underscore) format which
  `vivid` does not emit. Install `brew install coreutils` and use
  `gls`, or use [`eza`](../eza/) / `lsd` which speak `LS_COLORS`
  natively.
- **Theme + database are decoupled at the wrong moment if you
  rename categories.** If you rename `source.cpp` → `source.cxx`
  in the filetype database, every theme that referenced
  `source.cpp` silently loses that colour rule. `vivid` will not
  warn — diff the generated `LS_COLORS` before and after.
- **8-colour terminals get visibly worse output.** The bundled
  themes are designed for 24-bit colour; on `TERM=xterm` (8-colour)
  many categories collapse to the same closest-match. Pass
  `-m 8-bit` to opt into the 256-colour mapping table; for true
  8-colour terminals, write a minimal theme by hand.
- **Some shells / completion systems cache `LS_COLORS`.** zsh's
  `_files` completion reads the variable at startup; if you change
  it mid-session, run `compinit` or open a new shell to see the
  effect.
