# xonsh

## Overview

`xonsh` is a Python-powered cross-platform shell where every command line is
also valid Python. You can pipe processes the way `bash` does
(`du -sh * | sort -h | tail`), drop into Python expressions inline
(`ls @(sorted(glob.glob('*.log'), key=os.path.getsize)[-3:])`), and freely
mix the two without ever escaping into `python -c '...'`. Subprocess mode
(`$(...)` to capture, `!(...)` to get a `CommandPipeline` object,
`![...]` to stream straight to the terminal) is first-class alongside
the Python AST, so a one-off "find the 5 newest files modified by user
`bojun` and gzip them" goes from "a 30-line awk pipeline I have to debug
with `set -x`" to "a list comprehension and a `for f in ...: !(gzip
@(f))` loop with real exception handling".

It comes with a tab-completion engine that understands Python (objects,
attributes, kwargs) and shell (commands on `$PATH`, file paths, git
subcommands, `man` pages), `xontrib` plugins for the things that would
otherwise require a config-file dialect (prompt themes, vox virtualenv
manager, direnv-shaped per-directory env vars, fzf history, jedi
completions), and a `xonfig` introspection command that prints the live
configuration so "why did this not load my alias" is answerable in one
command instead of three `echo $PS1` calls.

## Repo URL

https://github.com/xonsh/xonsh

## Version

0.23.4 (released 2026-05-03)

## License

BSD 2-Clause — upstream LICENSE file:
[`LICENSE`](https://github.com/xonsh/xonsh/blob/main/LICENSE).

## Install

Homebrew (macOS / Linux):

```bash
brew install xonsh
```

pipx (recommended for isolated install):

```bash
pipx install 'xonsh[full]'    # bundles ptk-prompt + jedi + pyperclip extras
```

pip (into the current Python env):

```bash
pip install --user 'xonsh[full]'
```

AppImage (no Python required on the host):

```bash
curl -Lo xonsh "https://github.com/xonsh/xonsh/releases/download/0.23.4/xonsh-x86_64.AppImage"
chmod +x xonsh && ./xonsh
```

Verify:

```bash
xonsh --version    # xonsh/0.23.4
```

To make it a login shell:

```bash
command -v xonsh | sudo tee -a /etc/shells
chsh -s "$(command -v xonsh)"
```

Config lives at `~/.xonshrc` (Python source). A minimal one:

```python
$XONSH_SHOW_TRACEBACK = True
$AUTO_PUSHD = True
xontrib load coreutils vox jedi prompt_starship
```

## Why use it

Three things `xonsh` does that `bash` / `zsh` / `fish` do not, that
matter for an LLM-CLI / data-engineering workflow:

1. **Python in subprocess substitution.** `ls @(sorted(my_files, key=len))`
   passes a real Python list as separate `argv` entries with no quoting
   bugs; `bash`'s answer (`ls $(printf '%s\n' "${arr[@]}" | sort -k...)`)
   is fragile around spaces and newlines.
2. **Real exception handling on shell commands.** `try: !(curl -fsS
   $URL)` plus `except subprocess.CalledProcessError as e:` lets a
   one-off shell loop recover from per-iteration HTTP failures without
   `set -e` / `trap` gymnastics. The exit-code-as-int (`$?`) and
   exit-status-as-bool (`$(cmd) and ...`) idioms both work.
3. **`xontrib` plugin model.** Plugins are normal Python packages
   (`pip install xontrib-foo` → `xontrib load foo` in `.xonshrc`), so
   "add a fzf history picker" or "auto-activate this directory's
   virtualenv" is one line, and the plugin can import any Python
   library it needs instead of being a 200-line shell function.

For a workflow that already lives in Python (data wrangling, ML training
loops, agent harnesses), `xonsh` removes the impedance mismatch between
"the shell I run my scripts from" and "the language those scripts are
written in" — you can paste a working shell pipeline into a `.py` file
or vice versa and it keeps working.

## Vs Already Cataloged

- **Vs [`nushell`](../nushell/):** orthogonal philosophies. `nushell`
  invents a typed-pipeline DSL where every command emits structured
  data (tables, records); `xonsh` keeps `bash`-shaped byte streams and
  adds Python on top. Pick `nushell` when "treat `ps` output as a
  table and SQL-ify it" is the killer feature; pick `xonsh` when "I
  already think in Python" is.
- **Vs [`starship`](../starship/):** orthogonal — `starship` is a
  cross-shell prompt renderer, `xonsh` is a shell. They compose:
  `xontrib load prompt_starship` makes `xonsh` use the `starship`
  binary to render `PROMPT`.
- **Vs [`atuin`](../atuin/):** orthogonal — `atuin` is a SQLite-backed
  shell-history database with sync; works inside `xonsh` via
  `xontrib-atuin`.

## Caveats

- **Startup cost.** `xonsh` boots a CPython interpreter, so a cold
  shell is ~150-300 ms vs `bash`'s ~10 ms. For interactive use
  imperceptible; for scripts spawning a subshell per loop iteration,
  use `bash -c '...'` or `python -c '...'` directly.
- **Not POSIX.** Scripts that rely on `bash`-isms (`<<<` here-strings,
  `$(<file)`, parameter expansion forms like `${var//pattern/repl}`)
  do not run unmodified. `xonsh` provides equivalents (`f.read()`,
  `var.replace(...)`) but the porting work is real for non-trivial
  `.sh` files. Keep `#!/usr/bin/env bash` shebangs on existing scripts
  and only use `xonsh` for new code or interactive sessions.
- **Tooling assumes POSIX.** Some build tools (autoconf, certain
  `Makefile` recipes that shell out to `$SHELL`) misbehave when
  `$SHELL=xonsh`. Override per-tool (`SHELL=bash make ...`) or keep
  `bash` as the system `$SHELL` and run `xonsh` only as the
  interactive shell.
