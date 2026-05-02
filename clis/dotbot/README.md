# dotbot

> **Dotfile bootstrapper driven by a single YAML / JSON plan file.**
> Declares "link these files into `$HOME`, create these dirs,
> run these shell commands" and applies the plan idempotently.
> Bundled into your dotfiles repo as a git submodule so a fresh
> machine is one `git clone && ./install` away from a working
> shell.
> Pinned to **v1.24.0**
> ([LICENSE.md](https://github.com/anishathalye/dotbot/blob/master/LICENSE.md),
> MIT).

Source: <https://github.com/anishathalye/dotbot>

## TL;DR

`dotbot` is a ~1k-line Python script that reads an `install.conf.yaml`
from your dotfiles repo and executes its directives in order:

- `link:` — create symlinks (with options for `force`, `relink`,
  `create` parent dirs, glob patterns, `if` shell-condition guards)
- `create:` — `mkdir -p` for directories the symlinks need
- `shell:` — run arbitrary shell commands (with `stdout`,
  `stderr`, `stdin`, `quiet`, `description` controls)
- `clean:` — remove dead symlinks pointing into the dotfiles repo

It is *vendored* into your dotfiles repo as a submodule (`git
submodule add https://github.com/anishathalye/dotbot`) plus a
1-line `./install` wrapper, so `git clone --recursive
~/dotfiles && cd ~/dotfiles && ./install` is the entire
bootstrap on a fresh machine — no `pip install`, no system
dependency beyond Python 3 and git.

## Install

`dotbot` is **not** installed system-wide; it is vendored:

```bash
# In your existing dotfiles git repo
cd ~/dotfiles
git submodule add https://github.com/anishathalye/dotbot
git config -f .gitmodules submodule.dotbot.ignore dirty
cp dotbot/tools/git-submodule/install ./install
chmod +x ./install
git add . && git commit -m "Add dotbot bootstrap"

# Pin to v1.24.0
cd dotbot && git checkout v1.24.0 && cd ..
git add dotbot && git commit -m "Pin dotbot to v1.24.0"

# On a fresh machine
git clone --recursive git@github.com:you/dotfiles.git ~/dotfiles
cd ~/dotfiles && ./install
```

There is also `pipx install dotbot` if you prefer a system
install for one-off use, but the submodule pattern is the
canonical workflow.

## License

MIT — see
[LICENSE.md](https://github.com/anishathalye/dotbot/blob/master/LICENSE.md).
Permissive, embed-friendly. Vendoring `dotbot` as a git
submodule inside your own dotfiles repo (which is the intended
use) is fine under MIT regardless of how your dotfiles repo
itself is licensed.

## One Concrete Example

```yaml
# ~/dotfiles/install.conf.yaml
- defaults:
    link:
      relink: true
      create: true
      force: false

- clean: ['~']

- create:
    - ~/.config
    - ~/.local/bin
    - ~/projects

- link:
    ~/.zshrc: zsh/zshrc
    ~/.zshenv: zsh/zshenv
    ~/.gitconfig: git/gitconfig
    ~/.config/nvim:
      path: nvim
      create: true
    ~/.ssh/config:
      path: ssh/config
      mode: 0600
    ~/.config/foo/*:
      glob: true
      path: foo/*

- shell:
    - command: git -C ~/.tmux/plugins/tpm pull || git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
      description: Install / update tmux plugin manager
    - command: chsh -s "$(which zsh)"
      stdin: true
      stdout: true
      stderr: true
      description: Set zsh as login shell
      if: '[ "$SHELL" != "$(which zsh)" ]'
```

```bash
# Apply the plan
./install
# Reads install.conf.yaml, executes link/create/clean/shell in order,
# prints PASS/FAIL per directive. Idempotent: running it again on a
# converged machine is a no-op (relink=true sees existing correct
# links and skips them).

# Use a different config (e.g. work vs personal machine)
./install -c install.work.conf.yaml

# Verbose to debug a failing directive
./install -v
```

## Niche It Fills

**The "declarative dotfiles bootstrap, no global install" gap.**
Hand-written shell installers grow into 200-line bash scripts
with `if [ -L ~/.zshrc ] then rm` ladders and silent failures.
Heavier tools (`chezmoi`, `home-manager`) require a system
install before you can install your dotfiles — chicken-and-egg
on a fresh machine. `dotbot` lives *inside* your dotfiles repo
as a submodule, so the only system requirement is Python 3 and
git, both already present on every developer machine. The plan
file is YAML you can read in one screen, and the directive
ordering is the install order — there is no DAG to learn.

## Why use it

Three things `dotbot` does that the obvious alternatives don't:

1. **Self-contained bootstrap.** `git clone --recursive +
   ./install` is the entire onboarding. No "first install
   homebrew, then install ansible, then…". Recovering a wiped
   machine over a slow tethered connection is the test case
   this is optimised for.
2. **Plan order = execution order.** The YAML is read top-to-
   bottom and applied in sequence. No implicit DAG, no
   declarative ordering surprises. If `link:` needs a directory
   from `create:`, put `create:` first — the same way you'd
   write the bash script.
3. **`if:` guards are shell.** Conditional directives use a
   plain shell expression (`if: '[ "$(uname)" = Darwin ]'`).
   No DSL, no Jinja, no per-tool template language to memorise.
   Anything you'd write in bash you can put in `if:`.

For an LLM-CLI workflow, `dotbot` is the **per-host config
materialiser**: an agent provisioning a new shell sandbox can
`git clone --recursive` a curated dotfiles repo and `./install`
to land a known-good environment (shell, editor config, git
identity, ssh known_hosts) in one command, with a YAML diff that
a reviewer can read instead of a bash script that they can't.

## Vs Already Cataloged

- **Vs [`chezmoi`](../chezmoi/):** `chezmoi` is a single Go
  binary with templating, secret encryption, machine-specific
  data, an internal state directory, and a real DSL. Heavier
  but more powerful. Pick `chezmoi` when you need per-host
  templating or secret management; pick `dotbot` when you want
  the install plan to be reviewable YAML and the runtime to be
  "stock Python".
- **Vs [`yadm`](../yadm/):** `yadm` makes `$HOME` itself a git
  working tree (no symlinks, no manifest). Different model.
  `dotbot` keeps your dotfiles in a separate repo and *projects*
  them into `$HOME` via symlinks, so `git status` in `$HOME` is
  empty.
- **Vs `stow` ([`stow`](../stow/)):** `stow` is *only* the
  symlink farm — no shell hooks, no `mkdir`, no `if` guards,
  no submodule story. `dotbot` is "stow + create + shell + clean
  + plan file". Pick `stow` for pure dotfiles with no per-machine
  steps; pick `dotbot` when you need post-link shell (install
  tpm, set login shell, generate ssh keys).
- **Vs ansible / chef / saltstack:** Real config-management
  tools assume a control-plane, an inventory, and idempotent
  modules. Massive overkill for one developer's laptop. `dotbot`
  is the "it's just my machine, leave me alone" answer.

## Caveats

- **Python 3 is a runtime dep.** ~99 % of the time it's already
  installed; on a stripped Alpine container it isn't, and you
  need `apk add python3` first. Plan accordingly for ultra-
  minimal targets.
- **No rollback.** If a `shell:` directive halfway through the
  plan fails, prior `link:` operations stay applied. The intended
  recovery is "fix the YAML and re-run" (idempotent), not
  "transactional undo". Keep destructive steps near the end.
- **No per-host data files.** `dotbot` reads exactly one config
  per invocation. The convention for multi-host setups is one
  `install.<host>.conf.yaml` per host plus a tiny `./install`
  wrapper that picks the right one — *you* implement the
  selection logic, `dotbot` does not.
- **Submodule UX is the standard Git submodule UX.** Forgetting
  `--recursive` on `git clone` leaves an empty `dotbot/`
  directory and a confusing "no such file" error. The README
  walks through the canonical setup.
- **Slow release cadence is the point.** v1.24.0 (late 2024)
  and the version-1 series have been stable since 2014. The
  format is intentionally frozen, so old `install.conf.yaml`
  files keep working. Don't expect new features; expect "still
  works ten years later" to keep being true.
