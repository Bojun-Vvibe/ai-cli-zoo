# sheldon

> **Fast, configurable shell-plugin manager driven by a single
> declarative TOML file.** One Rust binary that clones / fetches
> plugins from GitHub / GitLab / arbitrary git remotes / raw URLs /
> local paths, generates the appropriate `source` lines, and caches
> the result so shell startup pays the cost of `eval "$(sheldon
> source)"` once per config change instead of every login. Pinned
> to **0.8.5** (commit
> `344dfc75e8c043c0d853007d2bdc00c85a74dbcf`),
> dual-licensed
> Apache-2.0 OR MIT
> ([LICENSE-APACHE](https://github.com/rossmacarthur/sheldon/blob/trunk/LICENSE-APACHE),
> [LICENSE-MIT](https://github.com/rossmacarthur/sheldon/blob/trunk/LICENSE-MIT)).

- **Repo:** https://github.com/rossmacarthur/sheldon
- **Latest version:** 0.8.5
- **License:** Apache-2.0 OR MIT (dual; SPDX `Apache-2.0 OR MIT`,
  `LICENSE-APACHE` + `LICENSE-MIT` at repo root)
- **Category:** `shell` / `plugins` / `dotfiles`
- **Language:** Rust

## What it does

`sheldon` reads `~/.config/sheldon/plugins.toml` — a flat declarative
table of plugin entries — and emits a single shell script (the "lock
file") that the shell rc sources on startup. A plugin entry is one
of: a GitHub repo (`github = "owner/name"`), a generic git remote
(`git = "https://…"`), a raw URL to a single file (`remote = "…"`),
or a local path (`local = "/abs/path"`). Each entry can pin a `tag`,
`branch`, or `rev`, declare which files to source via
`use = ["{{ name }}.plugin.zsh"]` Handlebars-templated globs (the
default templates already cover the common Oh-My-Zsh / Prezto
shapes), apply `apply = ["defer", "fpath"]` post-processors (defer
loading via `zsh-defer`, append directories to `$fpath`, etc.), and
gate by shell with `[plugins.foo]` only being included when
`SHELDON_SHELL` matches.

`sheldon lock` clones / pulls everything in parallel, regenerates
the lock script, and exits; `sheldon source` prints the generated
script to stdout (the canonical shell-rc one-liner is `eval
"$(sheldon source)"` which short-circuits to the cached lock file
when no plugin definitions changed). `sheldon add github
zsh-users/zsh-autosuggestions` mutates the config file from the CLI
so adding a plugin does not require opening an editor.
`sheldon edit`, `sheldon remove`, and `sheldon init --shell zsh`
round out the day-to-day surface. The whole thing is one ~6 MB
static Rust binary with no Python / Node / shell-script
prerequisites — appropriate for a minimal alpine container or an
SSH-only login that has no Oh-My-Zsh framework installed.

## When to pick it / when not to

Pick `sheldon` when you want a *plugin manager* that is configured
in version-controlled TOML rather than imperative shell code in
`.zshrc` / `.bashrc` — the `plugins.toml` lives next to your
[`chezmoi`](../chezmoi/) / [`yadm`](../yadm/) / [`stow`](../stow/)
managed dotfiles, diffs cleanly, and applies identically across
machines via `sheldon lock`. Pick it when shell startup latency
matters: the lock file is a single sourced script with no
per-plugin git work on every login (cold-start improvement is
order-of-magnitude vs naive `source ~/.oh-my-zsh/...` chains).
Pick it when you want a manager that is shell-agnostic — bash and
zsh are first-class, fish has community templates — rather than a
zsh-only framework. The Rust binary makes it the right answer in
distroless-style containers and on locked-down servers where
installing a Ruby (antibody) or Bash-script (zinit / antigen)
manager is friction.

Skip it if your dotfiles already work fine with vanilla `source`
lines and the rebuild cost is "minutes per year" — adding a plugin
manager has its own complexity. Skip it for fish if you are happy
with `fisher` ([`fisher`](../fisher/)) — fisher is fish-native
with first-class `funcsave` / `complete` integration that
`sheldon` matches but does not exceed for a fish-only workflow.
Skip it for nushell — `sheldon` does not target nushell's
module / overlay model. Skip it if you want the Oh-My-Zsh
*theme + plugin marketplace* experience: `sheldon` loads OMZ
plugins fine via its templates but ships none of the
discovery / curation that makes OMZ feel like a turnkey
distribution.

Vs already cataloged: orthogonal to [`starship`](../starship/) /
[`oh-my-posh`](../oh-my-posh/) — those are *prompt* renderers
loaded by `eval "$(starship init zsh)"`; `sheldon` is the *plugin
manager* that would source those `eval`s on your behalf alongside
auto-suggestions, syntax highlighting, completion bundles, and the
rest. Composes cleanly: a one-line `sheldon` plugin entry of
shape `[plugins.starship] inline = 'eval "$(starship init zsh)"'`
brings the prompt under the same lock-file workflow. Pairs with
[`fzf`](../fzf/) / [`atuin`](../atuin/) / [`zoxide`](../zoxide/)
— each ships an `init` snippet that becomes a one-line `inline`
or `remote` plugin entry. Pairs with [`chezmoi`](../chezmoi/) /
[`yadm`](../yadm/) for the dotfiles-as-VCS layer that owns the
`plugins.toml` itself.

## Example invocations

```bash
# First-time setup — generate a starter config for the current shell
sheldon init --shell zsh
# writes ~/.config/sheldon/plugins.toml with sensible defaults

# Add the canonical zsh-users plugins from the CLI (no editor needed)
sheldon add autosuggestions   github zsh-users/zsh-autosuggestions
sheldon add syntax-highlight  github zsh-users/zsh-syntax-highlighting
sheldon add completions       github zsh-users/zsh-completions
sheldon add async             github mafredri/zsh-async

# Pin a plugin to a tag for reproducibility across machines
sheldon add powerlevel10k github romkatv/powerlevel10k --tag v1.20.0

# Lock — clones everything in parallel, regenerates the cached script
sheldon lock --update

# The single line that goes in ~/.zshrc (after PATH, before prompt init)
eval "$(sheldon source)"

# Remove a plugin and regenerate the lock
sheldon remove autosuggestions
sheldon lock

# Edit the TOML directly when you need templates / apply chains
sheldon edit

# Verify
sheldon --version    # sheldon 0.8.5
```

A minimal `~/.config/sheldon/plugins.toml` after the above:

```toml
shell = "zsh"

[plugins.autosuggestions]
github = "zsh-users/zsh-autosuggestions"

[plugins.syntax-highlight]
github = "zsh-users/zsh-syntax-highlighting"
apply = ["defer"]   # load after the prompt to avoid blocking startup

[plugins.completions]
github = "zsh-users/zsh-completions"
apply = ["fpath"]   # add to $fpath instead of sourcing

[plugins.starship]
inline = 'eval "$(starship init zsh)"'
```

## Caveats

- **Lock file is a build artifact.** `sheldon lock` regenerates
  `~/.local/share/sheldon/plugins.lock` (or platform equivalent);
  do not check it into the dotfiles repo — the TOML is the source
  of truth, the lock is per-machine.
- **`sheldon source` is read-only fast-path.** It does not refresh
  remote plugins; run `sheldon lock --update` periodically (or
  pin everything to tags) to pull upstream changes intentionally.
- **Pre-1.0.** API and config schema have shifted historically;
  pin the binary version in dotfiles and read release notes
  before bumping.
- **No fish-native module loading.** Fish works via the same
  `source`-shaped templates but does not get fish's `funcsave` /
  `complete` ergonomics — use [`fisher`](../fisher/) for a
  fish-first plugin workflow.
