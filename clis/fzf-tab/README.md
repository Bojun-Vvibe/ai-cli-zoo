# fzf-tab

> **Replace zsh's tab completion menu with an interactive fzf
> picker** — a single-file zsh plugin that hooks into the standard
> `complete` widget and routes every completion list (file paths,
> git branches, kill PIDs, kubectl resources, ssh hosts, history
> words, anything `compsys` knows about) through `fzf`'s fuzzy
> finder, with previews. Pinned to **v1.3.0**
> ([LICENSE](https://github.com/Aloxaf/fzf-tab/blob/master/LICENSE),
> MIT).

Source: <https://github.com/Aloxaf/fzf-tab>

## TL;DR

`fzf-tab` is a zsh plugin (no binary, no daemon — one
`fzf-tab.plugin.zsh` file sourced from your `.zshrc`) that
intercepts the `Tab` key inside zsh and, instead of rendering the
default columnar completion menu, hands the candidate list to
`fzf` for fuzzy filtering and arrow-key selection. Because it
plugs in at the `compsys` layer, *every* zsh completion you
already have (built-ins, distro completions in `_*` files, plugin
completions from `oh-my-zsh` / `prezto`, the `_arguments` specs
shipped by `kubectl completion zsh`, `gh completion -s zsh`, etc.)
automatically gets fuzzy + preview behavior with no per-command
configuration.

## Install

```bash
# Prerequisites: zsh 5.4+ and fzf in PATH
brew install fzf zsh                   # macOS
sudo apt install zsh fzf               # Debian / Ubuntu

# Manual: clone and source from .zshrc
git clone https://github.com/Aloxaf/fzf-tab ~/.fzf-tab
echo 'source ~/.fzf-tab/fzf-tab.plugin.zsh' >> ~/.zshrc

# Or via a plugin manager (one of):
# zinit
zinit light Aloxaf/fzf-tab
# antidote / antigen
antigen bundle Aloxaf/fzf-tab
# zplug
zplug "Aloxaf/fzf-tab"
# oh-my-zsh: clone into $ZSH_CUSTOM/plugins and add to plugins=()
git clone https://github.com/Aloxaf/fzf-tab \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/plugins/fzf-tab"

# Pin to v1.3.0 explicitly:
git -C ~/.fzf-tab checkout v1.3.0

# IMPORTANT: source order matters. fzf-tab must come AFTER
# `compinit` and AFTER any other plugin that calls `compdef`
# (e.g. zsh-autosuggestions, zsh-syntax-highlighting are fine
# either side; oh-my-zsh's lib/completion.zsh runs compinit early).
```

Reload zsh (`exec zsh`) and hit `Tab` after any command — the
default menu is gone, replaced by an `fzf` overlay you can
substring-search.

## License

MIT — see
[LICENSE](https://github.com/Aloxaf/fzf-tab/blob/master/LICENSE).
Permissive, redistribute and modify freely; no attribution
required for binary distributions (there is no binary — it is
zsh source).

## One Concrete Example

```bash
# After install, every Tab works. A few high-leverage examples:

# 1. cd into a deeply nested dir by fuzzy substring
cd <Tab>           # fzf opens with every direct child; type "src api"
                   # to narrow to ./services/api-src/, Enter to fill in

# 2. checkout a git branch from 80+ remote branches
git checkout <Tab>     # fuzzy across local + remote branches

# 3. kill a process by name, with `ps` preview pane
kill <Tab>             # PID list with the process command line in
                       # the preview pane (configured below)

# 4. ssh to a host from ~/.ssh/config
ssh <Tab>              # fuzzy across Host blocks

# 5. kubectl resource picker (with the official zsh completion)
kubectl describe pod <Tab>   # fuzzy pod list, namespace-aware

# 6. enable file-content preview for any path completion
zstyle ':fzf-tab:complete:*:*' fzf-preview \
  '([[ -f $realpath ]] && bat --color=always $realpath) ||
   ([[ -d $realpath ]] && eza --tree --level=1 $realpath) ||
   echo $realpath'

# 7. group completions (file vs directory vs alias) by header
zstyle ':completion:*:descriptions' format '[%d]'
zstyle ':fzf-tab:complete:*:*' fzf-flags --height=50% --layout=reverse

# 8. accept on Enter, multi-select with Tab inside the picker
zstyle ':fzf-tab:*' continuous-trigger 'tab'
```

## Niche It Fills

**Universal fuzzy + preview layer for the zsh completion system.**
The native zsh menu is fast but list-only and exact-prefix; `fzf`
on its own is great but stand-alone (`fzf` doesn't know what
arguments `kubectl describe pod` accepts — `compsys` does).
`fzf-tab` is the glue: every existing `_*` completion definition
keeps working as the source of truth for *what* the candidates
are, and `fzf` becomes the surface for *how* you pick one. The
result is one keystroke (`Tab`) that does fuzzy + preview for
filesystem, git, ssh, kubectl, docker, npm scripts, env vars,
history words, and anything else `compinit` already knows about,
without writing per-command wrappers.

## Why use it

Three things `fzf-tab` does that the default menu does not:

1. **Substring + fuzzy match instead of prefix-only.** `cd <Tab>`
   followed by `api` matches `./services/api-src/` even though
   the directory does not start with `api`. Saves the "what was
   that directory called again" round-trip into `find` or
   `zoxide`.
2. **Inline preview pane per completion context.** Configure
   `fzf-preview` once per context (files, git branches, env vars,
   PIDs) and every relevant `Tab` shows the file contents, the
   commit log of the branch, the value of the variable, the
   process command line — without leaving the prompt or running
   a second command.
3. **Zero per-command setup.** Every distro / plugin / tool that
   ships a zsh completion automatically participates. Install
   `kubectl completion zsh`, `gh completion -s zsh`,
   `terraform -install-autocomplete` once, and the same
   fuzzy + preview behavior applies to all three the next time
   you hit `Tab`.

For an LLM-CLI workflow where an agent suggests a command to run
interactively, the human reviewer can `Tab`-cycle the suggested
arguments (paths, branches, pods) with fuzzy filtering and a
preview, instead of accepting the agent's exact spelling
verbatim.

## Vs Already Cataloged

- **Vs [`fzf`](../fzf/) (parent project):** `fzf` provides the
  fuzzy picker and the well-known shell key bindings (`Ctrl-T`
  for files, `Ctrl-R` for history, `Alt-C` for cd). Those are
  *separate* keybindings from `Tab`; they don't know what the
  current command expects. `fzf-tab` plugs `fzf` into the `Tab`
  key on top of the zsh completion system, so the picker is
  context-aware (kubectl pods after `kubectl describe pod`,
  branches after `git checkout`).
- **Vs [`carapace`](../carapace/):** carapace generates rich
  completion *specs* for hundreds of CLIs and ships them across
  bash / zsh / fish / pwsh / nu. It's complementary: install
  carapace as the spec source, install `fzf-tab` as the rendering
  layer in zsh, and the carapace specs flow through `fzf` on
  `Tab`.
- **Vs [`zoxide`](../zoxide/) / [`fzf`'s `Alt-C`](../fzf/):** both
  are directory jumpers tied to their own keybinding, not to
  `cd <Tab>`. `fzf-tab` makes the literal `cd <Tab>` workflow
  fuzzy without changing the verb you type, which lower the
  switching cost for humans who already type `cd`.
- **Vs `zsh-autocomplete` (not cataloged):** an alternative
  approach that auto-shows completions as you type (no `Tab`
  required) and uses `fzf` for some surfaces. More invasive,
  changes prompt rendering. `fzf-tab` is the conservative pick:
  only intercepts `Tab`, leaves typing latency untouched.

## Caveats

- **Zsh-only.** Bash, fish, nushell users need a different tool
  (`fzf`'s built-in keybindings, `carapace`, or shell-native
  completion).
- **Source order is load-bearing.** Must be sourced after
  `compinit` and after anything else that calls `compdef`. Wrong
  order silently disables the plugin. The README has the canonical
  ordering; with `oh-my-zsh`, it generally works because
  `compinit` runs in `lib/completion.zsh`.
- **Preview commands run on every keystroke inside the picker.**
  An expensive `fzf-preview` (e.g. `bat` on a 50 MB file, or
  `git log` against a huge repo) will add visible latency. Cap
  with `head -200` or use `--preview-window=hidden` and reveal on
  demand with `?`.
- **`compsys` quirks leak through.** If a particular CLI's zsh
  completion is buggy or returns the wrong candidates,
  `fzf-tab` faithfully forwards the bug — the fix is in the
  upstream `_<cmd>` file, not in fzf-tab.
- **No multi-select for most contexts.** Default behavior is
  single-pick on `Enter`. Multi-select with `Tab` inside the
  picker requires the `continuous-trigger` zstyle and only makes
  sense for completions that take a list (e.g. multi-file `rm`,
  multi-branch `git branch -d`).
- **Project pace.** v1.3.0 was tagged 2025-09; the plugin is
  mature and the surface is small (one file, one set of
  zstyles). Don't expect rapid feature churn — but don't expect
  abandonment either.
