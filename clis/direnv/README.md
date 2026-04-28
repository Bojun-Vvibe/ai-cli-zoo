# direnv

> **Per-directory environment variables loaded by your shell** —
> a single Go binary that hooks your shell prompt, looks for an
> `.envrc` file walking up from `$PWD`, and exports/un-exports
> variables so each project gets its own env without polluting
> the global shell. Pinned to **v2.37.1** (2025-07-20 release,
> [LICENSE](https://github.com/direnv/direnv/blob/master/LICENSE),
> MIT).

Source: <https://github.com/direnv/direnv>

## TL;DR

`direnv` is the replacement for `source ~/work/proj/env.sh`
discipline. Drop a file called `.envrc` in any project root with
`export DATABASE_URL=...` (and friends), `direnv allow`
once, and from then on `cd` into that tree auto-loads the env;
`cd` out unloads it. The shell hook (one line in `.zshrc` /
`.bashrc`) is the only setup. The killer feature is the
**stdlib** of `.envrc` helpers: `use flake`, `use nix`, `use
node 22`, `layout python 3.12`, `dotenv .env.local`,
`PATH_add ./bin` — declarative one-liners that compose with
`nvm` / `pyenv` / `nix` / `mise` without writing per-project
shell.

## Install

```bash
# Homebrew (macOS / Linux)
brew install direnv

# Linux package managers
# Debian/Ubuntu: apt install direnv
# Arch:          pacman -S direnv
# Fedora:        dnf install direnv
# Alpine:        apk add direnv
# Nix:           nix-env -iA nixpkgs.direnv

# Go
go install github.com/direnv/direnv/v2@latest

# install.sh (any OS)
curl -sfL https://direnv.net/install.sh | bash

# verify
direnv version    # 2.37.1

# Add the shell hook — pick one (this is what makes auto-load work):
# ~/.zshrc
eval "$(direnv hook zsh)"
# ~/.bashrc
eval "$(direnv hook bash)"
# ~/.config/fish/config.fish
direnv hook fish | source
# tcsh:  eval `direnv hook tcsh`
# elvish:  eval (direnv hook elvish | slurp)
```

After the hook, drop `.envrc` files in your projects and run
`direnv allow` once per file (an audit step — `direnv` refuses
to load any `.envrc` it has not seen approved, so an attacker
who plants one in a cloned repo cannot exfiltrate via
auto-execution).

## License

MIT — see
[LICENSE](https://github.com/direnv/direnv/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. minimal .envrc — set a project-local var
cat > ~/work/myproj/.envrc <<'EOF'
export DATABASE_URL="postgres://localhost/myproj_dev"
export OPENAI_API_KEY="$(pass api/openai)"
PATH_add ./bin
EOF
direnv allow ~/work/myproj/.envrc

# now: cd ~/work/myproj sets DATABASE_URL + adds ./bin to PATH;
#      cd .. unsets them.

# 2. load a .env.local file (12-factor style)
echo 'dotenv .env.local' > .envrc
direnv allow

# 3. per-project Python venv (auto-created, auto-activated)
cat > .envrc <<'EOF'
layout python python3.12
EOF
direnv allow
# .direnv/python-3.12/ is created and activated on cd-in

# 4. per-project Node version (delegates to nvm / fnm / asdf / mise)
cat > .envrc <<'EOF'
use node 22
EOF
direnv allow

# 5. nix flake project — load the dev shell automatically
echo 'use flake' > .envrc
direnv allow
# Nix devShell is loaded on cd-in, unloaded on cd-out

# 6. inspect what direnv would do without applying it
direnv status

# 7. revoke trust if .envrc looks suspicious
direnv deny

# 8. use direnv non-interactively (CI, scripts, agents)
eval "$(direnv export bash)"   # exports current dir's env to *this* shell
```

## Niche It Fills

**Per-directory env without each project rewriting the
"activate" script.** Every language-stack tool ships some
flavour of "source this": `source venv/bin/activate`, `nvm use`,
`pyenv shell`, `conda activate`. `direnv` is the *meta-layer*
that runs the right one for you on `cd` and undoes it on `cd
..`. The win is uniform: one shell hook, N projects, M
languages.

## Why use it

Three things `direnv` does that bare-shell `source ...` does not:

1. **Auto-unload on `cd` out.** The biggest practical win — env
   leakage between projects (left-over `DATABASE_URL` pointing
   at the wrong DB, stale `PYTHONPATH`) just stops happening.
   The shell hook tracks which `.envrc` is loaded and unloads
   exactly its exports when you leave.
2. **Trust-on-first-use audit (`direnv allow`).** Every `.envrc`
   must be explicitly approved before it can run, and re-edits
   re-prompt. This makes `git clone <repo>` safe even when the
   repo ships an `.envrc` — `direnv` will refuse to load it
   until you read it and approve it. The audit log lives in
   `~/.local/share/direnv/allow/`.
3. **`stdlib` helpers compose with the rest of the toolchain.**
   `use flake` for Nix, `use node 22` for nvm/fnm/asdf/mise,
   `layout python 3.12` for venvs, `dotenv` for 12-factor files
   — declarative one-liners that mean per-project `.envrc` files
   stay 3-5 lines instead of growing into bespoke scripts.

For an LLM-CLI workflow where an agent runs commands in a
project tree, `direnv exec /path/to/proj <cmd>` runs `<cmd>`
under the project's env without needing the agent to source
anything — useful when the agent's shell is non-interactive and
the hook never fires.

## Vs Already Cataloged

- **Vs [`mise`](../mise/):** overlap and complementary — `mise`
  is a runtime version manager (asdf successor: `mise install
  python@3.12 node@22 go@1.23`), and it has its own per-directory
  env support via `[env]` blocks in `mise.toml`. Pick `mise`
  alone when "I want one tool for tool-versions + per-dir env";
  pick `direnv` when "env management is the goal and runtime
  installs are someone else's job", or run both — `use mise` is
  a shipped `direnv` stdlib helper that delegates the
  tool-version side cleanly.
- **Vs [`atuin`](../atuin/) / [`zoxide`](../zoxide/):**
  orthogonal — those are shell augmentations for *history* and
  *navigation*; `direnv` is for *environment*. The three install
  cleanly side-by-side and don't overlap.
- **Vs [`just`](../just/) / [`task`](../task/):** orthogonal —
  those run named commands; `direnv` sets the env those commands
  run *in*. Compose: a `justfile` recipe runs under whatever
  env `direnv` loaded for the project tree.
- **Vs hand-written `source env.sh`:** the closest peer — same
  end state (env loaded for one project), but you have to
  remember to source on `cd` in *and* unset on `cd` out, and a
  malicious `env.sh` from a cloned repo runs with no audit.
  `direnv` automates both the load/unload and the audit.

## Caveats

- **Shell hook is mandatory.** Without `eval "$(direnv hook
  ...)"` in your rc, `cd` does nothing — `.envrc` files just sit
  there. Easy to install, easy to forget on a new machine.
- **Non-interactive shells skip the hook.** `bash -c 'cmd'` /
  CI runners / `Makefile` recipes do not see the env unless you
  explicitly `direnv exec . cmd` or `eval "$(direnv export
  bash)"` first. This is by design (security) but trips up
  agents and CI scripts.
- **`.envrc` is shell, not data.** Anything in an `.envrc` runs
  as Bash on `cd` in. Powerful (you can compute `DATABASE_URL`
  from `pass`/`vault`/`op`) but means a typo can `rm -rf` —
  `dotenv .env` is the safer subset for plain `KEY=value` files.
- **Trust prompt becomes muscle-memory `direnv allow`.** Power
  users hit `direnv allow` on every prompt without reading;
  defeats the audit purpose. Consider `direnv allow --for-tag`
  workflows or read the diff (`direnv status` shows what
  changed) before approving.
- **Slow `.envrc` (e.g. `use flake` cold-start) blocks every
  prompt.** `direnv` runs the file synchronously on `cd`. For
  Nix flakes the first eval can take many seconds; use
  `nix-direnv` (`use flake` from the `nix-direnv` package) which
  caches the evaluated env and stays fast on subsequent `cd`s.
- **Windows support is via WSL only.** Native Windows shells
  (PowerShell / cmd.exe) are not supported targets.
