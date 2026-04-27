# gum

> **A tool for glamorous shell scripts.** A single Go binary from
> Charm that exposes the
> [Bubble Tea](https://github.com/charmbracelet/bubbletea) /
> [Lip Gloss](https://github.com/charmbracelet/lipgloss) widget set
> as standalone shell commands — `gum input`, `gum choose`, `gum
> confirm`, `gum spin`, `gum filter`, `gum table`, `gum write`,
> `gum format`, `gum style` — so plain bash scripts get rich TUI
> prompts (fuzzy filter, multi-select, password input, themed
> banners, spinners) without writing any Go or pulling a
> Python TUI dependency. Pinned to **v0.17.0**
> ([LICENSE](https://github.com/charmbracelet/gum/blob/main/LICENSE),
> MIT).

Source: <https://github.com/charmbracelet/gum>

## TL;DR

`gum` is the right answer when a shell script needs to ask the
human something and you do not want to use `read -p` (no
arrow-key history, no fuzzy filter, no mouse, no theme) or
hand-roll an `fzf` invocation per prompt. Each `gum` subcommand
is one widget that reads from stdin / args, renders a polished
TUI, and writes the result to stdout — pure Unix shape, drop-in
to any shell. `gum choose tea coffee` opens a single-select
list. `echo "$LONG_TEXT" | gum filter` is fuzzy-filter on stdin.
`gum spin --title "deploying..." -- ./deploy.sh` shows a
spinner while the wrapped command runs. `gum confirm "wipe
prod?" && rm -rf …` replaces the y/n loop. The Charm style
inherits across commands via `GUM_*` env vars or per-call flags
so the look stays consistent.

## Install

```bash
# Homebrew
brew install gum

# Go
go install github.com/charmbracelet/gum@latest

# Linux package managers
# Arch: pacman -S gum
# Debian/Ubuntu (Charm apt repo):
#   sudo mkdir -p /etc/apt/keyrings
#   curl -fsSL https://repo.charm.sh/apt/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/charm.gpg
#   echo "deb [signed-by=/etc/apt/keyrings/charm.gpg] https://repo.charm.sh/apt/ * *" | sudo tee /etc/apt/sources.list.d/charm.list
#   sudo apt update && sudo apt install gum
# Nix: nix-env -iA nixpkgs.gum

# Windows
# scoop install gum
# winget install charmbracelet.gum

# from a release tarball
curl -Lo gum.tar.gz "https://github.com/charmbracelet/gum/releases/download/v0.17.0/gum_0.17.0_Darwin_arm64.tar.gz"
tar xf gum.tar.gz
sudo install gum /usr/local/bin/

# verify
gum --version    # gum version v0.17.0
```

Zero config; everything is per-call flags. Theme overrides go
in env vars (`GUM_INPUT_PROMPT_FOREGROUND="#FF06B7"`,
`GUM_CHOOSE_CURSOR_FOREGROUND="green"`) sourced from your
shell's rc file if you want a consistent house style.

## License

MIT — see
[LICENSE](https://github.com/charmbracelet/gum/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. fuzzy-pick a git branch
BRANCH=$(git branch --format='%(refname:short)' | gum filter --placeholder "branch...")
git checkout "$BRANCH"

# 2. multi-select files to add
FILES=$(git status --porcelain | awk '{print $2}' | gum choose --no-limit)
[[ -n "$FILES" ]] && git add $FILES

# 3. typed commit message with header / body split
TYPE=$(gum choose "feat" "fix" "docs" "refactor" "test" "chore")
SCOPE=$(gum input --placeholder "scope (optional)")
SUMMARY=$(gum input --placeholder "summary" --width 72)
BODY=$(gum write --placeholder "details (Ctrl+D when done)")
HEADER="${TYPE}${SCOPE:+($SCOPE)}: ${SUMMARY}"
gum confirm "commit as '$HEADER'?" && git commit -m "$HEADER" -m "$BODY"

# 4. spinner around a long-running command
gum spin --spinner dot --title "running tests..." -- cargo test --all

# 5. password input (no echo)
TOKEN=$(gum input --password --placeholder "API token")

# 6. themed banner / error message
gum style \
    --foreground 212 --border-foreground 212 --border double \
    --align center --width 50 --margin "1 2" --padding "1 2" \
    "Deploy complete" "$(date)"

# 7. render markdown inline (no separate `glow` install needed for short bits)
gum format -- "## Summary" "" "- 3 files changed" "- 0 tests broken"

# 8. table of structured data
printf 'name,size,modified\n%s\n' "$(ls -l | awk 'NR>1 {print $9","$5","$6" "$7}')" \
    | gum table --separator ","
```

## Niche It Fills

**Polished interactive prompts in shell scripts without leaving
shell.** Every onboarding script, deploy script, ad-hoc CLI
wrapper eventually grows a `read -p "are you sure? [y/N] "`
loop, an `fzf` invocation for fuzzy pick, and ANSI escape
codes for color. `gum` collapses all three into named
subcommands with consistent styling, so the script stays bash
(no Python TUI lib, no Node CLI framework) but feels like a
real product.

## Why use it

Three things `gum` does that ad-hoc shell does not, that pay
back the switching cost:

1. **Every widget is its own subcommand with stdin / stdout
   contract.** `gum filter`, `gum choose`, `gum input`, `gum
   write`, `gum confirm`, `gum spin`, `gum table`, `gum
   format`, `gum style`, `gum file` (file picker), `gum pager`
   — each reads from stdin / argv and writes to stdout, so
   they compose with shell pipes the way Unix commands always
   have. Nothing to learn beyond "what's the flag for
   placeholder text?".
2. **Theme inheritance via env vars.** Set `GUM_*` once in
   `.zshrc` (or in a `gum.env` you `source` per project) and
   every `gum` call across every script in the repo picks up
   the same colors / borders / cursors. House style without a
   config file format.
3. **No language lock-in.** Calls from bash, zsh, fish, nu,
   PowerShell, Makefiles, justfiles, Python `subprocess`, Go
   `exec.Cmd` — all fine, because the contract is "argv + env
   in, exit code + stdout out". The script that uses `gum` is
   still a portable shell script, not a TUI app rewrite.

For an LLM-CLI workflow, `gum` is what turns the human-in-the-
loop confirmation step from a janky `read -r REPLY` into a
real picker. A wrapper script around an agent CLI can `gum
choose` the model, `gum input --password` the API key, `gum
confirm` the destructive action, and `gum spin` while the
agent thinks — five lines of shell, looks like a product.

## Vs Already Cataloged

- **Vs [`fzf`](https://github.com/junegunn/fzf) (not cataloged):**
  closest peer for the filter / choose use case. `fzf` is the
  industry standard fuzzy-finder with a deeper feature set
  (preview windows, multi-key bindings, history-mode integration
  with shells). `gum filter` is simpler, themed, and ships
  alongside `gum choose` / `confirm` / `input` / `spin` /
  `style` as one binary — pick `gum` when you want a coherent
  widget set; pick `fzf` when fuzzy-filter alone is the whole
  job and you want every keybind it offers.
- **Vs [`mods`](../mods/) / [`glow`](../glow/) (Charm
  siblings):** orthogonal. `mods` is "LLM as Unix filter",
  `glow` is "render markdown to terminal", `gum` is "build
  pretty interactive prompts". They share the Charm look, and
  a typical Charm-stack script uses `gum input` to get the
  question, `mods` to run the LLM call, `glow` to render the
  reply.
- **Vs `whiptail` / `dialog` (not cataloged):** older curses
  prompt builders that fill the same niche; ugly by modern
  standards, awkward stdout vs stderr conventions (results go
  to stderr by default), and no fuzzy filter. `gum` is the
  Charm-era replacement.
- **Vs writing a Bubble Tea program in Go directly:** if you
  need a multi-screen TUI app with persistent state, write
  Bubble Tea. `gum` is the right answer for "one prompt at a
  time inside a shell script".

## Caveats

- **One widget per process.** `gum` is not a persistent TUI
  framework — each call is a fresh process. For a multi-screen
  stateful UI (think `lazygit`-shaped) you want Bubble Tea
  proper, not a chain of `gum` invocations.
- **Stdin must be a TTY for the interactive widgets.** Inside
  a non-interactive CI runner `gum input` / `gum choose` will
  fail or hang; gate calls with `[[ -t 0 ]] && gum confirm ||
  true` patterns or read the env (`CI=1`) and skip.
- **Theming via env vars is per-widget.** Each subcommand has
  its own `GUM_<SUBCOMMAND>_<FIELD>_<COLOR>` namespace
  (`GUM_INPUT_PROMPT_FOREGROUND`, `GUM_CHOOSE_CURSOR_*`,
  `GUM_CONFIRM_PROMPT_*`); there is no single "theme file" you
  source. The Charm docs catalog the full grid; in practice
  you set six or seven and stop.
- **Spinner exit code is the wrapped command's exit code.**
  `gum spin -- ./deploy.sh` exits with whatever `deploy.sh`
  exited; great for `set -e` scripts, surprising if you
  expected `gum` itself to swallow failures. Use `gum spin --
  ./deploy.sh || gum style --foreground 196 "deploy failed"`
  for the explicit error path.
- **`gum format` markdown is intentionally minimal.** For
  fully styled long-form markdown rendering, pipe through
  [`glow`](../glow/) (also Charm). `gum format` is for short
  inline blocks where you want one binary, not the full
  renderer.
