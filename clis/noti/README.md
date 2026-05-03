# noti

> **Desktop / mobile / chat notification on long-command completion.**
> A single Go binary that watches a command (or a stream piped into
> it) and sends a notification when the command exits — native macOS
> Notification Center, Linux `notify-send`, Windows toast, plus
> opt-in remote backends (Slack, Pushover, Pushbullet, Telegram,
> BearyChat, simplepush, Zulip, mattermost-style webhooks). Pinned
> to **v3.8.0** (SPDX: `MIT`,
> [LICENSE](https://github.com/variadico/noti/blob/main/LICENSE)).

Source: <https://github.com/variadico/noti>

## TL;DR

`noti` answers one question — "is that long-running thing done yet?"
— without you watching the terminal. Three forms:

- **Wrap a command:** `noti make release`
- **Append to a finished command:** `make release; noti`
- **Pipe a stream and notify on EOF:** `make release | noti`

When the wrapped command exits, `noti` fires a notification with the
exit status, runtime, and command line — locally on the desktop by
default, optionally also (or only) to Slack / Pushover / Telegram /
Pushbullet / Zulip / BearyChat / a generic webhook.

The whole program is one ~6 MB Go binary. No daemon, no plugin
manager, no per-shell hook needed (though a `precmd` hook for
auto-notify on commands over N seconds is one shell function — see
"Concrete example" below).

## Install

```sh
# macOS via Homebrew
brew install noti

# Any platform via Go
go install github.com/variadico/noti@v3.8.0

# Or grab a release tarball:
# https://github.com/variadico/noti/releases/tag/3.8.0
```

Verify:

```sh
noti -v
# noti version 3.8.0
```

Linux note — `noti` shells out to `notify-send` for the desktop
backend; install `libnotify-bin` (Debian/Ubuntu) or `libnotify`
(Fedora/Arch). Without `notify-send` the desktop backend fails;
remote backends still work.

## License

MIT — unrestricted. Safe to wrap into shell aliases, ship inside
team `rcfile` repos, embed in CI runner images, or include in a
proprietary developer toolkit.

## Concrete example: notify on every command that takes >30s

The 80% use case is "tell me when *any* slow command finishes,
without me having to remember to type `noti` every time." Drop this
into `~/.zshrc`:

```zsh
# Threshold in seconds.
NOTI_THRESHOLD=30

preexec() {
  __NOTI_CMD="$1"
  __NOTI_START=$EPOCHSECONDS
}

precmd() {
  local exit_code=$?
  if [[ -n "$__NOTI_START" ]]; then
    local elapsed=$((EPOCHSECONDS - __NOTI_START))
    if (( elapsed >= NOTI_THRESHOLD )); then
      noti -t "$( ((exit_code==0)) && echo 'Done' || echo 'FAILED' ) (${elapsed}s)" \
           -m "$__NOTI_CMD"
    fi
    unset __NOTI_START __NOTI_CMD
  fi
}
```

Now any command that takes ≥30 seconds (npm install, terraform plan,
cargo build, a long ssh, a dataset download, a `gh pr checks --watch`)
fires a desktop notification with the exit status and runtime when
it finishes. The bash equivalent uses `trap DEBUG` + `PROMPT_COMMAND`
in the same shape.

For remote notifications (laptop closed, phone in pocket), set the
backend env var once:

```sh
export NOTI_DEFAULT="banner pushover"
export NOTI_PUSHOVER_TOKEN="..."
export NOTI_PUSHOVER_USER="..."
```

`banner` keeps the local desktop popup; `pushover` adds a phone
notification through the Pushover service. Same shape for `slack`
(`NOTI_SLACK_TOKEN` + `NOTI_SLACK_CHANNEL`), `telegram`
(`NOTI_TELEGRAM_TOKEN` + `NOTI_TELEGRAM_CHATID`), and the rest.

## Niche

`noti` covers the "long command finished, please tell me" gap that
sits between three other tools none of which solve it cleanly:

- **Terminal-bell-on-prompt** (`PROMPT_COMMAND='printf "\a"'`) —
  free, works everywhere, but a beep with no metadata and no remote
  delivery; the laptop has to be in the room.
- **`tmux` activity monitor** (`set -g monitor-activity on`) — works
  for "any output in any pane" but cannot distinguish "command done"
  from "command printing progress", and is tmux-only.
- **A full job runner** ([`pueue`](../pueue/), [`task`](../task/),
  [`just`](../just/) with a notification recipe) — the right answer
  when you want a queue + history + dependency graph; overkill when
  you want "ping me when *this one* is done."

`noti` is the focused primitive: one binary, one wrap-or-pipe call,
one notification, optional remote fan-out.

## Why use it

1. **Recover wall-clock minutes per day.** Every long command you
   walk away from and remember to check 5 minutes after it actually
   finished is wasted time. A notification reclaims those minutes
   without requiring discipline.
2. **Multi-channel out of the box.** The same `noti` invocation goes
   to the desktop, the phone, a Slack channel, a Telegram bot, a
   webhook — set `NOTI_DEFAULT` once, every wrapped command lights
   up every backend you configured.
3. **Exit-status aware.** The notification text differs on success
   vs failure (or you template it yourself with `-t` / `-m`), so you
   can leave the room and know from the lock-screen banner whether
   you need to come back and triage.
4. **Pipe semantics work.** `long-job | noti` fires when the pipe
   closes — useful for streaming subcommands (`gh pr checks --watch`,
   `kubectl logs -f`, `docker compose up --abort-on-container-exit`)
   where the "done" signal is EOF on stdout, not a process exit.
5. **No daemon, no state.** `noti` is invoked, fires once, exits.
   There is nothing to keep running, nothing to crash, nothing to
   restart after a reboot.

## Vs already cataloged

- **vs [`pueue`](../pueue/)** — pueue is a *job queue* with history,
  dependencies, parallelism limits, and a TUI; `noti` is a
  *notification primitive*. Use pueue when you want to enqueue 50
  builds and walk away; use `noti` when you want one running command
  to ping you when it finishes. They compose: `pueue add -- noti
  long-job` enqueues the job *and* notifies when it completes.
- **vs [`gum`](../gum/) `gum spin`** — `gum spin` shows a spinner
  while a command runs in the foreground; `noti` does not block and
  notifies after the fact. `gum` for interactive scripts, `noti` for
  walk-away commands.
- **vs [`bat`](../bat/) / [`fzf`](../fzf/) / other interactive
  TUIs** — orthogonal; those make foreground work pleasant, `noti`
  makes background work observable.
- **vs `terminal-notifier` (macOS only) / `notify-send` (Linux
  only)** — `noti` wraps both behind one cross-platform binary plus
  remote backends, so the same `~/.zshrc` works on a macOS laptop
  and a Linux workstation without `uname` switches.

## Caveats

- **Remote backend tokens are bearer credentials.** Treat
  `NOTI_PUSHOVER_TOKEN` / `NOTI_SLACK_TOKEN` / `NOTI_TELEGRAM_TOKEN`
  like any other secret — store them in your shell-secret manager,
  not in a checked-in dotfiles repo. A leaked Slack token with
  channel-write scope can spam a workspace.
- **Linux desktop backend depends on `notify-send`.** On a headless
  server (CI runner, remote dev box), the `banner` backend has no
  display to write to — use the remote backends (`pushover`,
  `slack`, etc.) instead, or `ssh` the notification through to your
  laptop with a wrapper script.
- **macOS Notification Center silently drops banners** if the user
  has set the terminal app to "None" in System Settings →
  Notifications. First-time users on macOS occasionally think `noti`
  is broken when they have actually denied notification permission
  to the terminal.
- **Notifications are fire-and-forget.** `noti` does not retry on
  network failure for remote backends — a flaky wifi during the
  notification call means the ping is lost. Pair with a remote
  backend that has its own server-side queue (Pushover does;
  webhooks delivered to your own collector can) for best-effort
  retries.
- **Last release v3.8.0 is March 2025.** The repository is still
  active on GitHub (most recent commits 2026-05). The CLI surface
  has been stable across 3.x, so the slow release cadence reflects
  feature completeness rather than abandonment — pin `v3.8.0` and
  re-evaluate annually.

## How `noti` fits the LLM-CLI workflow

- **Long agent runs:** wrap the agent invocation —
  `noti opencode run "do the migration"` — and the desktop pings
  when the agent finishes (success or failure), so you can context-
  switch to other work without losing the moment of "agent is done,
  go review the diff."
- **Eval batches:** `noti python -m my_eval --suite full` for
  multi-hour evaluation runs that you cannot watch in real time.
- **Model downloads:** `noti ollama pull <large-model>` for the
  multi-GB initial pulls.
- **CI watching:** `gh pr checks --watch | noti` fires when the
  whole PR's checks reach a terminal state, so you can leave the PR
  page and get pinged when it goes green or red.

The composition pattern is always the same: anything that takes long
enough that you would otherwise context-switch *and forget* is the
right candidate for a `noti` wrap.
