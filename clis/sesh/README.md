# sesh

> **Smart tmux session manager** — a single Go binary that unifies
> tmux sessions, zoxide directories, tmuxinator/tmuxp configs, and
> arbitrary "startup" scripts behind one fuzzy picker, so jumping
> into the right project takes one keypress instead of "list →
> attach → cd → reopen editor". Pinned to **v2.26.1** (released
> 2025, [LICENSE](https://github.com/joshmedeski/sesh/blob/main/LICENSE),
> MIT).

Source: <https://github.com/joshmedeski/sesh>

## TL;DR

`sesh` sits on top of tmux and turns "session management" from a
mental tax into a fzf prompt. It enumerates: existing tmux
sessions, recent zoxide directories, hand-defined startup configs
(in `~/.config/sesh/sesh.toml`), and tmuxinator / tmuxp project
files — then lets you fuzzy-pick one. If the session exists it
attaches; if not, it creates the session in the right working
directory, runs the configured startup command (open `nvim`,
split a pane for `lazygit`, etc.), and attaches. The result is
that "switch to project X" becomes a single binding (`<prefix>+s
→ type "x" → enter`) regardless of whether X already has a
running session, has a project file, or is just a directory you
visited yesterday.

It is deliberately *not* a tmux replacement and not a
tmuxinator replacement — it is the *picker* layer that finally
makes those tools usable without remembering names.

## Why it's interesting

The "tmux session per project" workflow is decades old, but
every prior solution forces you to commit: tmuxinator demands a
yaml per project, raw tmux demands you remember session names,
zoxide gets you to the directory but not into a session. `sesh`
fuses all four sources behind one picker and adds session
*lifecycle* hooks (startup script, last-session toggle, kill
from the picker). It is the missing 50 lines of glue that the
tmux ecosystem never shipped.

## Install

```bash
# macOS / Linux
brew install joshmedeski/sesh/sesh

# Go
go install github.com/joshmedeski/sesh@latest

# verify
sesh --version    # 2.26.1
```

Recommended tmux binding (in `~/.tmux.conf`):

```tmux
bind-key "T" run-shell "sesh connect \"$(
  sesh list -i | fzf-tmux -p 80%,70% \
    --no-sort --ansi --border-label ' sesh ' --prompt '⚡  '
)\""
```

## Examples

```bash
# list everything sesh can switch to (sessions + zoxide + configs)
sesh list

# connect (create-or-attach) a session named after the directory
sesh connect ~/code/my-project

# jump to the last-used session (toggle)
sesh last

# define a startup config in ~/.config/sesh/sesh.toml
cat <<'EOF' >> ~/.config/sesh/sesh.toml
[[session]]
name = "dotfiles"
path = "~/.dotfiles"
startup_command = "nvim ."
EOF

# kill a session from the picker
sesh list | fzf | xargs sesh kill
```

## Use when

- You already live in tmux and waste time on `tmux ls` →
  `tmux attach -t name` → `cd project` → reopen editor.
- You want one keystroke to land in the right project with
  the right tools running, without writing a tmuxinator yaml
  for every directory.
- You use zoxide and want "frecent directories" to be valid
  jump targets for tmux sessions, not just `cd`.
- You are building a "project switcher" workflow on top of a
  tiling terminal (wezterm, kitty, ghostty) and want a single
  picker that handles both the create-session and
  attach-session cases.

Skip `sesh` if you do not use tmux at all (it is a tmux-first
tool — there is a wezterm backend but it is secondary), or if
you only ever work in one long-lived session and never switch
context.
