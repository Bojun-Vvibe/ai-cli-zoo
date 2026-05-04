# fisher

> **The plugin manager for the [fish](../fish-shell/) shell, written in
> ~250 lines of fish itself.** Resolves any GitHub repo (or local path,
> or gist, or git URL) that contains a fish-shaped layout (`functions/`,
> `completions/`, `conf.d/`) into the appropriate `~/.config/fish/`
> subdirectories, tracks the installed set in
> `~/.config/fish/fish_plugins`, and exposes four verbs (`install`,
> `update`, `remove`, `list`) — that's the entire surface. No daemon,
> no Lua runtime, no node_modules, no config DSL: the plugin file is
> what fish loads. Pinned to **4.4.8** (released 2024, MIT,
> [LICENSE.md](https://github.com/jorgebucaran/fisher/blob/main/LICENSE.md)).

Source: <https://github.com/jorgebucaran/fisher>

## Repo

- URL: <https://github.com/jorgebucaran/fisher>
- Owner: jorgebucaran (Jorge Bucaran — also author of `hyperapp`,
  `getopts.fish`; long-stable individual maintainer with ~9 years of
  fisher releases)
- License file:
  [LICENSE.md](https://github.com/jorgebucaran/fisher/blob/main/LICENSE.md)

## Version

`4.4.8` — verify with `fisher --version`. The 4.x line stabilised the
plugin format (no more `init.fish` / `key_bindings.fish` magic — fish's
own `functions/` + `completions/` + `conf.d/` autoload is the contract)
so a plugin written for `fisher 4.0` works with `fisher 4.4.8`. Updates
are rare and additive.

## License

**MIT** — permissive. Vendor, fork, embed, ship; no obligations.

## Install

One line in any fish session:

```fish
curl -sL https://raw.githubusercontent.com/jorgebucaran/fisher/main/functions/fisher.fish \
  | source && fisher install jorgebucaran/fisher
```

(yes — fisher installs itself; the bootstrap `source`s the function
once, then the function calls itself with `install jorgebucaran/fisher`
to make the install persistent under `~/.config/fish/functions/`).

Alternatively: `brew install fisher` on macOS / Linuxbrew, or copy
`functions/fisher.fish` into `~/.config/fish/functions/` by hand.

## What it does

`fisher install <plugin>` resolves a plugin spec — `owner/repo` (latest
default branch), `owner/repo@tag` (pinned), `owner/repo@branch`,
`/abs/local/path`, or any `git`-cloneable URL — fetches its tree, and
copies files into `~/.config/fish/` by their parent directory:

- `functions/foo.fish` → autoloaded function `foo` (lazy-loaded on
  first call; zero shell-startup cost for unused plugins)
- `completions/foo.fish` → tab-completion definitions for command `foo`
- `conf.d/foo.fish` → ran at fish startup (the place a plugin sets
  abbreviations, key bindings, event handlers, prompt segments)

Plugin state lives in `~/.config/fish/fish_plugins` — one plugin spec
per line, checked into your dotfiles repo. `fisher update` reconciles
the installed set against this file (bisync: installs missing,
removes stale, upgrades to declared tags), so a `git pull && fisher
update` on a new machine reproduces the exact plugin set deterministic-
ally — the same property [`zinit`](#) / [`antigen`](#) / [`oh-my-zsh`](#)
provide for zsh but with a one-page mental model.

`fisher remove <plugin>` calls each removed file's `_uninstall` event
handler (fish's native event system) before deleting, so a plugin that
set abbreviations or key bindings can clean up after itself — the
shell state after `fisher remove` matches the state before
`fisher install`.

The whole tool is one fish file (~250 lines), readable in one sitting —
the most auditable package manager any shell ships with.

## When to use

- **You use [`fish`](../fish-shell/) and want a deterministic, dotfile-
  trackable plugin set.** `fish_plugins` is the lock-file equivalent;
  commit it to your dotfiles repo and any new machine bootstraps to the
  identical plugin state with `fisher update`.
- **You want lazy-loaded fish completions for tools that don't ship
  them.** Plugins like `jorgebucaran/autopair.fish`,
  `PatrickF1/fzf.fish`, `meaningful-ooo/sponge`,
  `jorgebucaran/nvm.fish`, `franciscolourenco/done` are one
  `fisher install` away and contribute zero startup cost when unused.
- **You compose with [`starship`](../starship/) /
  [`oh-my-posh`](../oh-my-posh/) for the prompt.** The prompt is its
  own binary; fisher manages the *rest* of the fish customisation
  (functions, abbrs, completions, key bindings) without conflict.
- **You want the smallest possible plugin manager surface.** No DSL,
  no manifest format beyond a flat list, no network beyond `git fetch`,
  no telemetry. The 250-line script is the entire trust boundary.

## When NOT to use

- **You don't use fish.** fisher is fish-only by design; the plugin
  format depends on fish's `functions/` / `completions/` / `conf.d/`
  autoload semantics. Use [`zinit`](#) /
  [`oh-my-zsh`](https://ohmyz.sh) / `antidote` / `znap` for zsh,
  `vim-plug` / `lazy.nvim` / `packer.nvim` for Neovim,
  `fisher`-style is not portable.
- **You want a curated mega-distro experience.** `oh-my-fish` and the
  legacy `fundle` were attempts at the OMZ-style "1000 plugins, 200
  themes, install one bundle". fisher is deliberately the opposite —
  one verb per action, you pick the plugins. If the curation is the
  value, OMF is the wrong layer to compete with.
- **You distribute non-fish-shaped configuration via a repo and want
  fisher to install it.** A plugin must follow the
  `functions/`/`completions/`/`conf.d/` layout. Repos that ship a
  single `init.fish` you must `source` manually are not fisher
  plugins; fisher will install them but they won't autoload.
- **You need plugin sandboxing / capability isolation.** A fisher
  plugin is plain fish code that runs in your interactive shell —
  same privileges as your `~/.config/fish/config.fish`. Audit before
  installing; fisher does not isolate.

## AI-native angle

- **`fish_plugins` is a one-line shell-environment manifest an agent
  can produce.** A "set up my new dev box" agent emits the plugin
  list (often: `jorgebucaran/nvm.fish`, `jorgebucaran/autopair.fish`,
  `PatrickF1/fzf.fish`, `meaningful-ooo/sponge`, `IlanCosman/tide` or
  `starship/starship` for prompt) and `fisher update` reproduces it
  deterministically — no per-machine drift.
- **Pairs with [`starship`](../starship/) for one-config-everywhere
  prompts.** starship handles the prompt across shells; fisher
  handles the fish-specific completions + abbreviations layer above
  it. Two manifests (`starship.toml` + `fish_plugins`) reproduce a
  fully-furnished fish session.
- **Pairs with [`fzf`](../fzf/) via `PatrickF1/fzf.fish`** to bind
  `Ctrl+R` to fuzzy history search, `Ctrl+T` to fuzzy file insert,
  `Ctrl+Alt+L` to fuzzy git log — the same `fzf` keybindings the
  bash/zsh user has, packaged correctly for fish.
- **Pairs with [`atuin`](../atuin/)** which has first-class fish
  installation via a `conf.d/` snippet and integrates cleanly into a
  fisher-managed plugin set.

## Alternatives in this catalog

- [`fish-shell`](../fish-shell/) — the shell fisher manages plugins
  for. Fisher is to fish what `lazy.nvim` is to Neovim: a thin
  package manager built around the editor/shell's native autoload
  semantics.
- [`mise`](../mise/) / [`fnm`](../fnm/) / [`proto`](../proto/) —
  language-version managers that ship fish integration. Use these
  for *language toolchain* version management; use fisher for
  *shell function / completion* plugins.
- [`starship`](../starship/) / [`oh-my-posh`](../oh-my-posh/) — cross-
  shell prompts. Orthogonal: prompt is one binary called from
  `fish_prompt`; fisher does not manage prompts directly (though the
  fish-native `IlanCosman/tide` is fisher-installable).
- [`fzf`](../fzf/) / [`zoxide`](../zoxide/) / [`atuin`](../atuin/) —
  the shell-augmentation classics. Each has a small fish integration
  snippet that fisher-installable plugins (`PatrickF1/fzf.fish`,
  `kidonng/zoxide.fish`, atuin's own `init fish`) wrap correctly.
- [`pkgx`](../pkgx/) / [`flox`](../flox/) — system-level package
  managers that don't compete: pkgx/flox install the *binaries*
  (`fzf`, `starship`), fisher installs the *fish glue* that wires
  them into the shell.
