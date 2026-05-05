# tmuxinator

> **Project-scoped tmux session manager** that turns a single
> YAML file into a reproducible window / pane layout — start a
> 4-window, 9-pane workspace with one command.
> Pinned to **v3.3.8** (SPDX: `MIT`,
> [LICENSE](https://github.com/tmuxinator/tmuxinator/blob/master/LICENSE)).

Source: <https://github.com/tmuxinator/tmuxinator>

## TL;DR

`tmuxinator` is a Ruby gem that reads
`~/.config/tmuxinator/<project>.yml` and translates it into the
exact sequence of `tmux new-session` / `split-window` /
`send-keys` calls needed to recreate a complete project
workspace — windows named for what they do, panes pre-laid-out
in the geometry you want, and the per-pane startup commands
already typed and executed. Run `mux <project>` (or `tmuxinator
start <project>`), and 200 ms later you are inside the same
session you closed yesterday: editor on the left, log tail
upper-right, dev server lower-right, REPL in window 2.

## Install

```bash
# Homebrew (macOS / Linux)
brew install tmuxinator

# Ruby (gem)
gem install tmuxinator

# verify
tmuxinator version    # 3.3.8
mux version           # alias installed by the gem
```

Requires `tmux >= 1.9` on `PATH` and `$EDITOR` set (used by
`tmuxinator new <project>` to open a new project file in your
editor).

## License

MIT — see
[LICENSE](https://github.com/tmuxinator/tmuxinator/blob/master/LICENSE).

## One Concrete Example

```bash
# 1. scaffold a new project file and open it in $EDITOR
mux new myapp

# 2. minimal project YAML
cat > ~/.config/tmuxinator/myapp.yml <<'YAML'
name: myapp
root: ~/code/myapp

windows:
  - editor:
      layout: main-vertical
      panes:
        - vim .
        - git status
        - tail -f log/development.log
  - server: bundle exec rails server
  - db:
      layout: even-horizontal
      panes:
        - bundle exec rails console
        - psql myapp_development
YAML

# 3. start (or attach to) the session
mux start myapp
# equivalent: tmuxinator start myapp

# 4. enumerate / edit / delete projects
mux list
mux edit myapp
mux delete myapp

# 5. lint a project file before starting it (CI-friendly)
mux doctor
mux debug myapp     # prints the exact tmux commands it would run

# 6. start with tmuxinator's session-name override (parallel work)
mux start myapp --name myapp-pr-422

# 7. stop a running session by project name
mux stop myapp
```

## Niche It Fills

**Reproducible per-project tmux layouts as version-controlled
YAML.** Without `tmuxinator`, the workflow is: open tmux, split
the window, run `cd ~/code/myapp && vim`, switch pane, run
`cd ~/code/myapp && rails server`, switch window, … every
morning. With `tmuxinator`, the *layout* is data: one YAML file
per project, optionally checked into the project repo (or
dotfiles), and `mux <project>` reproduces it bit-for-bit.

## Why use it

1. **Layout-as-data.** A 9-pane workspace is 30 lines of YAML,
   not 30 muscle-memory keystrokes.
2. **Project-scoped, not session-scoped.** Each project gets a
   named, repeatable starting point — switching context is
   `mux stop oldproj && mux start newproj`, not "where did I
   leave that pane?"
3. **Composable with everything tmux already does.** Once
   started, the session is a normal tmux session — `tmux
   attach`, `tmux switch-client`, your existing `.tmux.conf`
   keybinds, `tmux-resurrect`, all still work.
4. **Idempotent attach.** `mux start <project>` attaches if a
   matching session already exists, creates it if not — safe
   to wire into shell startup.
5. **Lint + dry-run.** `mux doctor` checks for missing tmux /
   `$EDITOR`. `mux debug <project>` prints the exact tmux
   commands it would issue, useful for debugging a YAML that
   doesn't behave as expected.

## Vs Already Cataloged

- **Vs [`sesh`](../sesh/):** `sesh` is a fuzzy-finder over
  *existing* tmux sessions / git repos / zoxide entries — the
  "jump to a session" surface. `tmuxinator` is the "define what
  a session looks like" surface. They compose: define layouts
  in tmuxinator, jump between them with sesh.
- **Vs [`zellij`](../zellij/):** zellij is a tmux-class
  multiplexer with built-in per-session layouts via its own
  KDL config — pick zellij when you want one tool for both
  multiplexing and layout, pick tmux + tmuxinator when the
  rest of your tooling already speaks tmux.
- **Vs [`tmuxp`](../tmuxp/):** tmuxp is the Python sibling
  with the same model (YAML / JSON / Python session manifests).
  Pick tmuxp if your toolchain is Python-first; pick tmuxinator
  if it's Ruby-first or you prefer the more mature ecosystem
  (tmuxinator predates tmuxp by ~5 years).
- **Vs [`overmind`](../overmind/) / [`hivemind`](../hivemind/):**
  Those run a `Procfile` of long-lived processes inside a
  single foreground tmux/log surface — they answer "start the
  app's processes." tmuxinator answers "lay out my entire
  workspace including the editor, REPLs, and ad-hoc panes." Use
  both: a tmuxinator pane that runs `overmind start`.
- **Vs [`mprocs`](../mprocs/):** mprocs is a single-window TUI
  for a fixed set of processes — no tmux. Pick it when the
  workflow is "watch N processes," pick tmuxinator when the
  workflow is "be inside a workspace with editor + processes
  + REPL panes I'll interact with."

## Caveats

- **Ruby runtime required.** Adds ~30 MB and a `gem install`
  step on a fresh box — friction if your workflow is otherwise
  Ruby-free. Alternatives: `tmuxp` (Python) or
  [`zellij`](../zellij/) (single static binary).
- **YAML, not Lua / Ruby DSL.** Conditional layouts ("if I'm
  on a laptop screen, use a 2-pane layout, otherwise 4-pane")
  require ERB templates or shelling out — fine for the common
  case, awkward for highly dynamic workspaces.
- **No layout snapshotting.** `tmuxinator` defines layouts;
  it doesn't *capture* a layout you built manually back into
  YAML. For "save the current session," reach for
  `tmux-resurrect` / `tmux-continuum` (orthogonal — they
  persist *running* sessions across reboots).
- **`pre_window` runs in every pane.** Easy footgun: putting
  `cd ~/project` in `pre_window` is fine, putting `bundle
  install` in `pre_window` will run it once per pane.
