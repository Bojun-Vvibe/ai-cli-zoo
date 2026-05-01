# tmuxp

> **Declarative tmux session manager** — describe a multi-window,
> multi-pane tmux session in a YAML / JSON file (windows, layouts,
> panes, per-pane `start_directory`, per-pane `shell_command` lists,
> environment, hooks) and `tmuxp load my-project.yaml` reconstructs
> it deterministically — attaching if a matching session already
> exists, freezing a live session back into a file with `tmuxp
> freeze <name>`, and converting between the legacy
> [`tmuxinator`](https://github.com/tmuxinator/tmuxinator) and
> [`teamocil`](https://github.com/remi/teamocil) formats with
> `tmuxp convert`. Pinned to **v1.67.0** (commit
> `64f7fe32b4de0bbeb091a31d80f4e7d53d2fcf5a`,
> [LICENSE](https://github.com/tmux-python/tmuxp/blob/master/LICENSE),
> MIT).

Source: <https://github.com/tmux-python/tmuxp>

## TL;DR

`tmuxp` is the Python answer to `tmuxinator` (Ruby) — same
declarative-session idea, no Ruby runtime. A workspace file
(`~/.config/tmuxp/myproject.yaml`) names a session, lists windows,
and gives each window an array of panes; each pane carries an
optional `start_directory` and an array of `shell_command` lines
that get sent on launch. `tmuxp load myproject` builds the whole
session at once, attaches you to it, and re-attaches on subsequent
runs without re-running the boot commands. The library underneath
is `libtmux` (also tmux-python), a pure-Python wrapper around the
`tmux` control-mode protocol — so the same workspace file is
re-usable from scripts that want to introspect or mutate a live
session without shelling out to `tmux ...`.

The killer features are (1) **`freeze`** — point it at a tmux
session you assembled by hand and it spits out the YAML that
recreates it, so the path from "I tweaked it interactively" to
"committed to the repo" is one command, and (2) **`convert`** —
it round-trips with `tmuxinator` / `teamocil` formats so a
mixed-language team does not have to re-author every workspace.

## Install

```bash
# pipx (recommended — isolated venv, single binary on PATH)
pipx install tmuxp

# pip (user)
pip install --user tmuxp

# Homebrew
brew install tmuxp

# verify
tmuxp --version    # 1.67.0
tmux -V            # tmuxp needs tmux >= 2.4 on PATH
```

Workspace files live in `~/.config/tmuxp/*.yaml` (or
`$TMUXP_CONFIGDIR`); a per-project `.tmuxp.yaml` at the repo root
is auto-discovered by `tmuxp load .`.

## One Concrete Example

```yaml
# .tmuxp.yaml — committed at the repo root
session_name: webapp
start_directory: ~/code/webapp
windows:
  - window_name: editor
    layout: main-vertical
    panes:
      - shell_command:
          - nvim
      - shell_command:
          - git status
  - window_name: server
    panes:
      - shell_command:
          - pyenv shell 3.12.5
          - uvicorn app:app --reload --port 8000
  - window_name: db
    panes:
      - shell_command:
          - docker compose up postgres
      - shell_command:
          - sleep 3
          - psql -h localhost -U app
  - window_name: tests
    panes:
      - shell_command:
          - pytest --watch -x -q
```

```bash
# bring the whole session up (or attach if it's already running)
tmuxp load .

# freeze a session you built interactively into a YAML you can commit
tmuxp freeze webapp > .tmuxp.yaml

# convert a tmuxinator file a teammate had
tmuxp convert ~/.tmuxinator/oldproj.yml > .tmuxp.yaml

# kill the whole session (rebuild from scratch on next load)
tmux kill-session -t webapp
```

## Niche It Fills

**Reproducible tmux session as a committed artifact.** A README
that says "open 4 panes, in pane 1 run X, in pane 2 run Y" is a
human-only contract; a `.tmuxp.yaml` at the repo root is a
machine-executable one. For an LLM-CLI workflow that wants to
spin up a deterministic dev environment (editor pane + server
pane + log-tail pane + agent-CLI pane) with one command, this
removes the per-machine drift.

## Vs Already Cataloged

- **Vs [`zellij`](../zellij/):** zellij is a *terminal multiplexer
  itself* with a layout file (`KDL`); it replaces tmux. tmuxp
  *manages tmux sessions* — pick zellij if you want a modern
  multiplexer with batteries included; pick tmuxp if you (or
  your team) are committed to tmux and want declarative session
  files on top of it.
- **Vs [`mprocs`](../mprocs/):** mprocs runs a fixed set of
  processes in a single TUI dashboard (one pane per process,
  no nesting, no detach-and-reattach across sessions). tmuxp
  builds *full tmux sessions* you can detach / SSH-reattach /
  share. Use mprocs for "watch these 5 processes during dev";
  use tmuxp for "this is the whole working environment."
- **Vs [`pueue`](../pueue/):** pueue is a background task queue
  (submit jobs, see their state, see their logs) — different
  layer entirely, no UI parallel.
- **Vs raw `tmux new-session ... \; split-window ... \;
  send-keys ...`:** the `tmux` shell-script approach works but
  is order-sensitive, hard to read, and does not survive a
  layout tweak. tmuxp's YAML is declarative and `freeze`-able.

## Caveats

- Requires `tmux >= 2.4` on PATH — on stale corporate Linux
  images you may need to install a newer `tmux` first.
- Python 3.9+ runtime; on hosts where Python is locked down,
  `pipx install tmuxp` into a user venv is the cleanest path.
- `freeze` captures *layout and running command lines* but
  cannot recover the original `start_directory` if the pane
  has since `cd`-ed elsewhere — review the generated YAML.
- The `shell_command` array sends keystrokes via `send-keys`
  by default; commands that read from stdin (interactive
  prompts) need `enter: false` + a follow-up entry, or the
  newer `shell_command_before` hook, otherwise they swallow
  the next line.
- Workspace files are **per-user** by default
  (`~/.config/tmuxp/`); committing `.tmuxp.yaml` at the repo
  root and using `tmuxp load .` is the team-shared pattern.
