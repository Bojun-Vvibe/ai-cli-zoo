# dotdrop

> **Dotfile manager that templates as it deploys** — a Python
> CLI that takes a single source repository of dotfiles and
> deploys them to many machines, where each machine sees a
> *different* rendered version: paths, package lists, colours,
> SSH hosts, and conditional blocks all flow through Jinja2
> templating gated by per-profile variables, so one repo
> simultaneously serves your laptop, your work box, your
> servers, and your container base image. Pinned to **v1.13.1**
> ([LICENSE](https://github.com/deadc0de6/dotdrop/blob/master/LICENSE),
> GPL-3.0-only).

Source: <https://github.com/deadc0de6/dotdrop>

## TL;DR

`dotdrop` is the dotfile manager for the case `stow` /
`chezmoi`-with-defaults / a hand-rolled `bootstrap.sh` does not
cover: *the same logical config file but different rendered
content per host*. The unit of work is a **config.yaml** that
maps **dotfiles** (paths in your repo) to **profiles** (named
hosts) with **variables** that the deploy step substitutes via
Jinja2. `dotdrop import ~/.config/nvim/init.lua` slurps a real
file into the repo and registers the mapping; `dotdrop install
-p laptop` materialises every dotfile mapped to the `laptop`
profile into your `$HOME` (or any `--workdir`), running each
through the templater so `{% if profile == "work" %}` blocks
expand differently per host. `dotdrop compare` is the inverse
diff — show what's drifted between the rendered repo and the
live filesystem — and `dotdrop update` pulls live edits back into
the repo with the templating reversed where possible. The whole
tool is one `pip install dotdrop` (or `pipx`) plus a Git repo of
your own.

## Install

```bash
# pipx (recommended — isolated venv)
pipx install dotdrop

# pip --user
pip install --user dotdrop

# Homebrew
brew install dotdrop

# Arch
yay -S dotdrop                         # AUR

# from source
git clone https://github.com/deadc0de6/dotdrop.git
cd dotdrop && pip install --user .

# verify
dotdrop --version    # 1.13.1
```

`dotdrop` itself is stateless; the source of truth is a Git repo
you control with one `config.yaml` and a `dotfiles/` tree. Bring
your own version control — typical setup is a `~/dotfiles` repo
with `dotdrop` as a submodule under `dotdrop/dotdrop/` so a fresh
machine bootstrap is `git clone --recurse-submodules <repo> &&
./dotdrop.sh install -p $(hostname -s)`.

## License

GPL-3.0-only — see
[LICENSE](https://github.com/deadc0de6/dotdrop/blob/master/LICENSE).
The deployed *content* (your dotfiles) keeps whatever licence you
ship; only the `dotdrop` binary itself is GPL-3.0-only, and that
binary is invoked from your shell, not embedded into a derivative
work, so the licence has no practical effect on your config repo.

## One Concrete Example

```bash
# 1. bootstrap: clone the upstream layout once
mkdir -p ~/dotfiles && cd ~/dotfiles
git init
git submodule add https://github.com/deadc0de6/dotdrop.git dotdrop
cp dotdrop/bootstrap.sh dotdrop.sh && chmod +x dotdrop.sh
mkdir dotfiles
cat > config.yaml <<'YAML'
config:
  backup: true
  banner: false
  create: true
  dotpath: dotfiles
  ignoreempty: true
  keepdot: true
  longkey: false
  workdir: ~/.config/dotdrop
profiles:
  laptop:
    dotfiles:
      - f_zshrc
      - f_gitconfig
      - f_nvim
    variables:
      git_email: me@personal.example
      pkg_extras: ["mpv", "yt-dlp"]
  workbox:
    dotfiles:
      - f_zshrc
      - f_gitconfig
      - f_nvim
    variables:
      git_email: me@employer.example
      pkg_extras: []
dotfiles:
  f_zshrc:
    src: zshrc
    dst: ~/.zshrc
  f_gitconfig:
    src: gitconfig
    dst: ~/.gitconfig
  f_nvim:
    src: nvim
    dst: ~/.config/nvim
YAML

# 2. import an existing file — dotdrop moves it into the repo
#    and replaces it with a placeholder mapping
./dotdrop.sh import -p laptop ~/.zshrc ~/.gitconfig ~/.config/nvim

# 3. add a Jinja2 conditional to the imported gitconfig
#    (dotfiles/gitconfig)
cat >> dotfiles/gitconfig <<'JINJA'
[user]
    email = {{@@ git_email @@}}
{%@@ if profile == "workbox" @@%}
[url "git@github.example:"]
    insteadOf = https://github.example/
{%@@ endif @@%}
JINJA

# 4. dry-run the deploy to see exactly what would change
./dotdrop.sh install -p laptop --dry

# 5. real deploy — backs up overwritten files to *.dotdropbak
./dotdrop.sh install -p laptop

# 6. on the work box, same repo, different rendered output
ssh workbox 'cd ~/dotfiles && ./dotdrop.sh install -p workbox'

# 7. live edits drifted from the repo? compare them
./dotdrop.sh compare -p laptop
# ==> diff against the rendered template, not the raw source

# 8. pull a live change back into the source (template-aware)
./dotdrop.sh update -p laptop ~/.zshrc

# 9. list everything dotdrop knows about
./dotdrop.sh files -p laptop          # what would deploy
./dotdrop.sh detail -p laptop         # full per-file metadata
./dotdrop.sh profiles                 # every profile defined
```

## Niche It Fills

**Dotfiles as a templated artifact, with profile = host.** The
common dotfile workflow is "one repo, one rendered output,
identical on every machine"; `stow` and the simpler use of
`chezmoi` / `yadm` / `rcm` deploy literal files. The moment you
have *two* hosts that need most of the same config but with
*different* paths / package sets / SSH endpoints / colours,
those tools force you into manual per-host overrides or pre-/
post-install scripts. `dotdrop` makes the variation a first-class
data model: profiles + variables + Jinja2 templates, with the
import / compare / update verbs that keep the live filesystem
and the repo in sync. For an LLM-CLI workflow that has the agent
manage its own runtime config across dev box, CI runner, and
container, the templated profile model is exactly the shape that
`agent-config-{dev,ci,container}.yaml` wants to be.

## Vs Already Cataloged

- **Vs [`chezmoi`](../chezmoi/):** the closest peer in scope and
  the one you should default to for *most* dotfile setups.
  `chezmoi` ships with built-in templating (Go's `text/template`),
  encrypted secrets via `gpg` / `age` / `keepassxc`, an
  opinionated source layout, and a single static Go binary with
  no Python runtime. `dotdrop` predates it and has the **profile
  model** as the centerpiece (chezmoi templates by `.chezmoi.toml`
  conditionals; dotdrop templates by named profiles you select
  with `-p`), plus a Python plugin model that lets you call
  arbitrary code in templates. Pick `chezmoi` when you want one
  static binary and encrypted-secret support out of the box; pick
  `dotdrop` when explicit named profiles + Jinja2 are a better
  fit for how you think about your hosts.
- **Vs [`yadm`](../yadm/):** `yadm` is "git for your dotfiles"
  with class-based alternates (`#H`, `#O`, `#C` host / OS / class
  suffixes that select which file wins). Lighter, no template
  language, no profile abstraction — the right pick when "git
  with a few suffix conventions" is enough. `dotdrop` is the
  next step up when the `#H` suffix tree starts duplicating
  whole files for one-line differences.
- **Vs [`stow`](https://www.gnu.org/software/stow/) (not yet
  cataloged):** `stow` is pure symlink farming — one source
  directory, one `stow vim` to symlink it into `$HOME`. No
  templating, no profiles, no per-host variation. The right
  pick if every machine gets the *literal same* file; the
  ceiling is the moment you need any per-host divergence.
- **Vs [`mise`](../mise/) / [`flox`](../flox/) /
  [`devbox`](../devbox/):** orthogonal — those manage *runtime
  toolchains* (which `node`, which `python`, which `terraform`).
  `dotdrop` manages *config files* (`.zshrc`, `.gitconfig`,
  `~/.config/nvim`). Most operators run both: a toolchain
  manager for binaries, a dotfile manager for the files those
  binaries read.
- **Vs [`ansible-pull`](https://docs.ansible.com/ansible/latest/cli/ansible-pull.html)
  / [`saltstack`] / [`puppet`] for "config management on a
  laptop":** those are full configuration-management systems
  designed for fleets of servers with a control plane and a
  module ecosystem. Massive overkill for a personal dotfile
  setup; the right answer when you are already running Ansible
  for your servers and don't want a second tool. `dotdrop`
  sits on the laptop / single-user side of that line.

## Caveats

- **GPL-3.0-only and Python-3 dependency.** The licence covers
  the binary itself, not the deployed content, so it doesn't
  contaminate your dotfile repo. The Python dependency means
  one `pipx install` per machine; the bootstrap-on-fresh-OS
  step needs Python 3 in PATH before `dotdrop` can run, which
  is fine on every modern distro and macOS but a real cost on
  busybox / Alpine-minimal where `chezmoi`'s static binary wins.
- **Jinja2 syntax inside config files needs escaping for
  languages that already use `{{ }}` / `{% %}`.** `dotdrop`
  defaults to `{{@@ var @@}}` / `{%@@ if ... @@%}` exactly to
  avoid colliding with shell, JSON, Helm, Liquid, etc. — pick
  a delimiter style and document it in the repo's README so
  future-you doesn't fight the template engine.
- **Profile selection is a flag, not a detection.** `dotdrop`
  doesn't auto-pick a profile from `$(hostname)` / `$(uname)` —
  you wrap `dotdrop install -p $(hostname -s)` in a shell
  alias or use the `dynvariables:` block to compute it. The
  upside is that the same `-p ci-runner` deploys reproducibly
  in any environment that sets the right hostname, including
  Docker / GitHub Actions; the downside is a one-line wrapper
  per machine on first setup.
- **Compare / update against templated files is best-effort.**
  Round-tripping `update` from a deployed file back into the
  templated source can lose template structure if the live edit
  changed text inside a `{% if %}` block — `dotdrop` warns and
  asks before clobbering. Treat `update` as "merge assistant",
  not "atomic reverse-template", and always review the resulting
  diff before committing.
- **Backups are local-file `*.dotdropbak`, not versioned.**
  Every overwritten file gets one `.dotdropbak` next to it; the
  *previous* `.dotdropbak` is overwritten on the next deploy.
  The real undo path is `git revert` in your dotfiles repo
  followed by `dotdrop install`, not the per-file backups.
- **No first-class secret management.** Encrypted-template
  support exists via the `actions:` block calling `gpg` /
  `age` / `pass` from a pre-install hook, but it's a wiring job,
  not a built-in like `chezmoi`'s. If your dotfiles include
  API keys or SSH private keys, design that wiring up front
  (or pick `chezmoi`).
