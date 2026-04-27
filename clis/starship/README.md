# starship

> **Cross-shell prompt that auto-detects project context** — a
> single Rust binary that renders your shell prompt in
> milliseconds, with first-class modules for git, language
> versions, cloud contexts, package metadata, and command
> duration. Pinned to **v1.25.0** (commit
> `457f16069b666d76e202708e0e3464122b57a6d5`,
> [LICENSE](https://github.com/starship/starship/blob/master/LICENSE),
> ISC).

Source: <https://github.com/starship/starship>

## TL;DR

`starship` replaces your shell's `PS1` with a fast, configurable
prompt that works identically across bash, zsh, fish, PowerShell,
nushell, ion, elvish, tcsh, and xonsh. It detects what is in the
current directory — `package.json` ⇒ Node version, `Cargo.toml`
⇒ Rust toolchain, `go.mod` ⇒ Go version, `pyproject.toml` ⇒
Python + active venv, `Dockerfile` ⇒ container context — and
shows a module per context, hiding it when irrelevant. Git
branch + dirty state + ahead/behind appear automatically inside
a repo. Configuration is one TOML file
(`~/.config/starship.toml`) shared by every shell. Render budget
is per-module timeout (default 500 ms) so a slow git status on
a giant repo never blocks the prompt.

## Install

```bash
# Homebrew (macOS / Linux)
brew install starship

# Cargo
cargo install --locked starship

# one-line installer (any OS)
curl -sS https://starship.rs/install.sh | sh

# Linux package managers
# Arch: pacman -S starship
# Nix: nix-env -iA nixpkgs.starship
# Debian / Ubuntu: snap install starship  (or use the installer above)

# Windows
# scoop install starship
# winget install Starship.Starship

# verify
starship --version    # starship 1.25.0
```

Then wire it into your shell (one line in your shell rc):

```bash
# ~/.zshrc
eval "$(starship init zsh)"
# ~/.bashrc
eval "$(starship init bash)"
# ~/.config/fish/config.fish
starship init fish | source
# PowerShell $PROFILE
Invoke-Expression (&starship init powershell)
```

For language version modules and icons, install a Nerd Font
(`brew install --cask font-jetbrains-mono-nerd-font`) and set
your terminal to use it.

## License

ISC — see
[LICENSE](https://github.com/starship/starship/blob/master/LICENSE).
Permissive (functionally equivalent to MIT, fewer words), no
attribution required for binaries.

## One Concrete Example

```bash
# 1. install + wire into zsh
brew install starship
echo 'eval "$(starship init zsh)"' >> ~/.zshrc
exec zsh

# 2. inside a Rust project, the prompt now reads:
#    ~/code/project on main [!?] via 🦀 v1.83.0
#    ❯
#    — branch + dirty + untracked + Rust toolchain, all auto-detected.

# 3. drop a config to tune it
mkdir -p ~/.config
starship preset nerd-font-symbols -o ~/.config/starship.toml

# 4. inspect what each module produces (debugging)
starship explain
starship module rust
starship module git_branch

# 5. measure render cost (should be <50 ms in steady state)
starship timings

# 6. transient prompt: keep recent prompts compact in history
#    ~/.config/starship.toml
#    [character]
#    success_symbol = "[❯](bold green)"
#    [fill]
#    symbol = " "

# 7. per-directory overrides for monorepos
#    ~/.config/starship.toml
#    [git_status]
#    disabled = false
#    [package]
#    disabled = true   # turn off package.json version in this repo

# 8. pre-cache prompt for faster cold starts
starship session
```

## Niche It Fills

**The shell-agnostic prompt.** If you switch between zsh on the
laptop and bash in a container and PowerShell on a Windows VM,
you previously maintained three prompt configs (oh-my-zsh,
bash-it, oh-my-posh). `starship` is one binary + one TOML file
that renders identically in all of them, with modules that hide
themselves when irrelevant — the prompt is empty in `/tmp`,
shows git+language+cloud in a project directory, and does it in
under 50 ms.

## Why use it

Three things `starship` does that hand-rolled `PS1` does not,
that pay back the switching cost:

1. **Auto-detection per directory.** Walk into a folder with
   `package.json` and the Node version module appears; walk
   into a sibling with `pyproject.toml` and the Python venv
   module appears instead. You do not configure "show node
   version when..." — you configure how the module *looks* if
   it triggers, and starship handles the *when*. ~50 modules
   cover the common languages, package managers, cloud
   contexts (`AWS_PROFILE`, GCP project, `KUBECONFIG`),
   container runtimes, and battery / job count.
2. **Per-module timeout, prompt never blocks.** Default 500 ms
   per module; if `git status` on a giant monorepo takes
   longer, the git module silently drops for that prompt
   render and the rest still draws. `starship timings` shows
   you which modules are expensive in your config, so you can
   tune (`scan_timeout`, `disabled = true` per slow module,
   or `command_timeout` global cap).
3. **Same prompt across every shell.** `starship init <shell>`
   emits the right hook for nine shells from one binary; the
   TOML config is shared. Move from zsh to fish to nushell
   without re-learning prompt syntax — the prompt looks
   identical, the cursor still goes in the same place.

The transient-prompt pattern (full prompt while editing, compact
prompt for history) keeps the scrollback readable on long
sessions; the `command_timeout` showing duration after slow
commands run replaces the "did that take 2 seconds or 2
minutes?" `time …` ritual.

## Vs Already Cataloged

- **Vs [`atuin`](../atuin/):** orthogonal — `atuin` replaces
  shell *history* (search, sync, stats); `starship` replaces
  shell *prompt* (what gets drawn before each command). Run
  both: `starship` for context awareness on the prompt,
  `atuin` for instant fuzzy history recall on `Ctrl-R`.
- **Vs [`zoxide`](../zoxide/):** orthogonal — `zoxide`
  replaces `cd` with a frecency-ranked jumper. Both improve
  the moments around running a command (`zoxide` to get to
  the right dir, `starship` to remind you what is in it),
  with no overlap.
- **Vs `oh-my-zsh` / `oh-my-posh` / `powerlevel10k` (not
  cataloged):** competing — those are zsh-only (or
  PowerShell-only for `oh-my-posh`) prompt frameworks.
  `powerlevel10k` is faster than `starship` in raw zsh
  benchmarks (instant prompt + lazy load) and has more
  zsh-specific polish; `starship` wins when you need the same
  prompt across multiple shells, when you want fewer moving
  parts (one binary vs a framework + plugins), or when your
  team sets up dev environments in heterogeneous shells.
- **Vs hand-rolled `PS1`:** raw `PS1` wins on zero-dependency
  portability (works on any bash, no install) and on absolute
  speed (no subprocess fork per render). `starship` wins on
  every other axis once you want git status, language
  versions, or cross-shell consistency.

## Caveats

- **Adds 10–50 ms per prompt render in steady state.** Not
  visible to humans typing, but in a tight `for f in *; do …;
  done` loop where each iteration triggers a prompt redraw,
  the overhead accumulates. `starship timings` quantifies it;
  disable expensive modules per-directory or globally if you
  hit a hot loop.
- **Modules can shell out and slow the prompt.** Language
  version modules call `node --version`, `python --version`,
  `rustc --version`, etc. on first render in a directory.
  Per-module `command_timeout` caps the damage; on cold
  network mounts or volumes with slow language toolchains,
  `disabled = true` for the offending module is the right
  call.
- **Nerd Font is effectively required.** Default presets use
  glyphs that render as tofu without a Nerd Font installed
  and selected in the terminal. The `starship preset
  plain-text-symbols` preset removes them, but the default
  appearance assumes you have one.
- **`oh-my-posh` (cataloged-adjacent) is the closest direct
  competitor on Windows.** If your fleet is mostly PowerShell,
  `oh-my-posh` integrates more deeply with Windows Terminal
  themes; `starship` is still the better cross-platform pick.
- **Init must come last in your shell rc.** If you set `$PS1`
  or `precmd`/`PROMPT_COMMAND` after `starship init`, you
  silently overwrite the prompt. Put `eval "$(starship init
  …)"` at the bottom of `~/.zshrc` / `~/.bashrc`.
- **TOML config has no schema validation in older versions.**
  Typos in module names or option keys silently no-op; check
  with `starship explain` after any config edit and use
  `starship config` to dump the effective config.
