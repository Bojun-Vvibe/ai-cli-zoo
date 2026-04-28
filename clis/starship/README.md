# starship

- **Repo:** https://github.com/starship/starship
- **Version:** v1.25.0 (latest stable, 2026-04)
- **License:** ISC ([LICENSE](https://github.com/starship/starship/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install starship` · `cargo install --locked starship` · `curl -sS https://starship.rs/install.sh | sh` · binary releases on the GitHub release page

## What it does

`starship` is a cross-shell prompt: one static Rust binary that renders
the dynamic part of your shell prompt (PS1) for bash, zsh, fish,
PowerShell, Nushell, Ion, Elvish, Tcsh, Xonsh, and Cmd, configured by a
single `~/.config/starship.toml`. The shell-side glue is two lines (one
`eval` of `starship init <shell>` in your rc file) and the rest is
declarative: ~70 built-in modules each detect a piece of context (current
git branch + dirty/ahead/behind, the language version of the project in
`pwd` for ~30 ecosystems including Rust/Go/Node/Python/Ruby/Java/.NET/
Elixir/Zig/Deno/Bun, the active Kubernetes context + namespace, the
active AWS / Azure / GCP profile, the active Nix shell, the
`direnv`-loaded env, the current Docker / Podman context, battery state,
exit code of the last command, command duration if it exceeded a
threshold, and so on) and the TOML lets you reorder them, restyle them,
hide them by default and show them only when relevant
(`disabled = true` per module, or `format = ""` to suppress), or
substitute custom symbols. Performance is the load-bearing claim: the
binary uses async detection, caches stat() results inside one prompt
render, and short-circuits modules that do not apply to the current
directory, so a typical `<10 ms` render budget holds even in monorepos
with deep `.git` directories. The init script also installs a
right-prompt (where supported), continuation-prompt, and
transient-prompt handler, so the previous command's prompt collapses to
a one-line marker after you press Enter — your scrollback stays
readable instead of being half prompt.

## When to pick it / when not to

Pick `starship` if your prompt currently lies to you (shows a stale git
branch because you `cd`'d into a worktree, shows the wrong Python
version because pyenv state did not propagate, shows nothing at all
about the active Kubernetes context so you `kubectl delete` in the
wrong cluster), or if you maintain dotfiles across more than one shell
and want the prompt definition to live in one TOML file instead of
forked-and-drifted bash / zsh / fish snippets. It is the default-good
choice for laptops where you regularly switch between language
ecosystems and cloud profiles in the same terminal session — the
context modules are where the value is, not the colours. Pair with
[`zoxide`](../zoxide/) for `cd` replacement (both are init-script
shell integrations and compose cleanly), with [`atuin`](../atuin/) for
shell-history search (atuin redraws its own UI but respects starship's
prompt format), with [`direnv`](../direnv/) for per-directory env (the
starship `direnv` module surfaces the loaded `.envrc` as a glyph), and
with [`mise`](../mise/) for runtime version management (the per-language
modules read mise's pinned version automatically).

Skip it on a hot-path SSH bastion or a constrained container shell
where you genuinely want a single-character `$` and zero subshells per
prompt — the rendering cost of `starship` is small but not zero, and
the value of cross-shell context modules collapses when you only ever
use one shell on that host. Skip it if your team has a hand-tuned
zsh-only prompt in a shared dotfiles repo that already covers the
context you need (powerlevel10k with the same module set is comparable
on zsh and uses the zsh async machinery natively). Skip the
language-version modules on a host where they would shell out to a
toolchain you have intentionally not installed — turn those modules off
explicitly rather than paying the "tool not found" timeout. And skip
`starship` if you object on principle to a binary that runs on every
prompt; the project's threat model is "your shell already runs your
shell, this is one more local exec," but that is a posture call.

## Example invocations

```bash
# One-line shell init (put in ~/.zshrc, ~/.bashrc, ~/.config/fish/config.fish)
eval "$(starship init zsh)"

# Print the effective config (after defaults + your overrides merge)
starship config

# Open the config file in $EDITOR; creates it if missing
starship configure

# Render the prompt once, for debugging
starship prompt

# Render the prompt with a fake last-command exit status, to test the
# error module styling without breaking a real command
starship prompt --status 1 --cmd-duration 4200

# Print every module that *would* render right now, with its timing
starship explain

# Time each module — surfaces the slow one in a deep monorepo
starship timings

# Print bash completion for starship itself
starship completions bash

# Bootstrap a new toml from the preset library
starship preset gruvbox-rainbow -o ~/.config/starship.toml
```

Sample `~/.config/starship.toml` for reference (terse, k8s + cloud + git aware):

```toml
add_newline = false
format = """
$directory\
$git_branch$git_status\
$kubernetes\
$aws\
$nodejs$python$rust$golang\
$cmd_duration\
$line_break$character"""

[directory]
truncation_length = 3
truncate_to_repo = true

[kubernetes]
disabled = false
format = '[$context(\($namespace\))]($style) '
style = "bold blue"

[aws]
format = '[$symbol($profile )]($style)'
symbol = "aws "

[cmd_duration]
min_time = 2_000   # only show if > 2s
format = "[$duration]($style) "
```
