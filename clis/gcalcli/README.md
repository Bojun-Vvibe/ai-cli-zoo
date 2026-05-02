# gcalcli

> **Google Calendar from the command line** — list agenda, add events,
> import `.ics`, run `remind` daemons, and dump search results in
> plain text or JSON without ever opening a browser tab.
> Pinned to **v4.5.1**
> ([LICENSE](https://github.com/insanum/gcalcli/blob/HEAD/LICENSE),
> MIT).

Source: <https://github.com/insanum/gcalcli>

## TL;DR

`gcalcli` is the longest-lived CLI front-end to Google Calendar
(first commit 2009, still actively maintained). It speaks the
Google Calendar v3 API directly, caches an OAuth refresh token in
`~/.gcalcli_oauth`, and exposes everything Calendar can do as
subcommands you can pipe into `awk`, `jq`, `fzf`, or a shell
script: `gcalcli agenda`, `gcalcli add`, `gcalcli search`,
`gcalcli edit`, `gcalcli delete`, `gcalcli import`, `gcalcli
remind`, `gcalcli quick "lunch with sam tomorrow noon"`. Output
modes include human-friendly colored TTY tables, plain text for
pipes, TSV, and JSON for downstream tools.

## Install

```bash
# pipx (recommended — isolated venv, all platforms)
pipx install gcalcli

# Homebrew
brew install gcalcli

# Linux distros
# Arch:           pacman -S gcalcli
# Debian/Ubuntu:  apt install gcalcli
# Fedora:         dnf install gcalcli
# Alpine:         apk add gcalcli
# Nix:            nix-env -iA nixpkgs.gcalcli

# verify
gcalcli --version    # 4.5.1

# first run will open a browser to authorize a Google account; the
# refresh token is stored at ~/.gcalcli_oauth and reused thereafter.
gcalcli list          # show calendars you have access to
```

You will need a Google Cloud project with the Calendar API
enabled and an OAuth client ID (the docs walk through the 6-click
setup); `gcalcli init` ships a guided flow.

## License

MIT — see
[LICENSE](https://github.com/insanum/gcalcli/blob/HEAD/LICENSE).
Permissive; embedding inside other CLIs or container images is
fine.

## One Concrete Example

```bash
# 1. agenda for the next 14 days, colored, in the terminal
gcalcli agenda today "in 14 days"

# 2. just today, plain text suitable for piping into a status bar
gcalcli --nocolor agenda today tomorrow | head -20

# 3. quick-add an event using natural language (Calendar parses it server-side)
gcalcli quick "design review thursday 3pm 45min in room K2"

# 4. structured add with reminders and a video conference
gcalcli add \
  --title "weekly 1:1" \
  --where "https://meet.example.test/abc-defg-hij" \
  --when "2026-05-04 10:00" --duration 30 \
  --description "rolling agenda doc: …" \
  --reminder "10 popup" --reminder "1440 email"

# 5. import an .ics file (e.g. a conference schedule)
gcalcli import ./kubecon-2026.ics

# 6. search and dump as TSV for jq-less pipelines
gcalcli search "design review" --tsv \
  | awk -F'\t' '{print $1, $4}'

# 7. long-running reminder daemon: pop a desktop notification 5 min before each event
gcalcli remind 5 'notify-send "calendar" "%s"'

# 8. delete an event by title (with confirmation prompt)
gcalcli delete "weekly 1:1"
```

## Niche It Fills

**The "calendar as a Unix datasource" gap.** Google Calendar is
where most people's actual day lives, but Calendar's web UI is
modal, mouse-first, and impossible to script. `gcalcli` turns the
calendar into something `cron`, shell scripts, status bars, and —
increasingly — LLM agents can read and write. There is no other
maintained, full-coverage CLI for Google Calendar; the
alternatives are either browser extensions, half-finished Go
prototypes, or `khal` + `vdirsyncer` (which is great, but is a
separate local-CalDAV stack rather than a thin client).

## Why use it

Three concrete things `gcalcli` makes possible that the web UI
does not:

1. **Agenda in your shell prompt / status bar / tmux.** `gcalcli
   --nocolor agenda today tomorrow | head -3` is fast enough
   (cached OAuth token, single HTTPS round-trip) to run from a
   `tmux` `status-right` every minute, so "next meeting in 12
   min" is always on screen without a browser tab.
2. **Bulk import / export as a real workflow.** Conference
   schedules, oncall rotations, and class timetables ship as
   `.ics`; `gcalcli import` ingests them in one command. `gcalcli
   agenda --tsv > week.tsv` gives you a snapshot you can diff
   week-over-week.
3. **Scriptable reminders that aren't Calendar's reminders.**
   `gcalcli remind 5 'say "%s"'` (macOS) or `… 'notify-send …'`
   (Linux) lets you wire calendar events into any local action —
   mute Slack, dim lights via Home Assistant, start a recording —
   none of which Calendar's built-in reminder system can do.

For an LLM-CLI workflow, `gcalcli agenda --tsv today "in 7 days"`
is a one-line "what is on my schedule" tool the agent can call
before proposing a meeting time, and `gcalcli quick "$NL_PHRASE"`
is a one-line "create the meeting we just agreed on" tool. Both
return immediately and produce machine-checkable output, which
makes them safer to expose to an autonomous loop than the
Calendar REST API directly.

## Vs Already Cataloged

- **Vs [`khal`](../khal/) + [`vdirsyncer`](../vdirsyncer/):**
  `khal` is a beautiful local TUI calendar that reads/writes
  iCalendar files in `~/.local/share/calendars`; `vdirsyncer`
  syncs those to/from CalDAV (including Google via app
  passwords). That stack is the right answer if you want to live
  fully offline, mix providers, or hate Google. `gcalcli` is the
  right answer if you want a single thin command that talks
  straight to Google with no local sync layer to break, and
  output that pipes cleanly into other tools.
- **Vs [`taskwarrior`](../taskwarrior/):** Taskwarrior is a
  todo-list engine, not a calendar — there is no concept of
  "this event happens at 14:00 in room K2 with these
  attendees". For tasks, use Taskwarrior; for time-blocked
  events with invitees, use `gcalcli`. Many people run both.
- **Vs `curl` against the Calendar v3 API directly:** raw API
  use means rolling your own OAuth dance, your own pagination,
  your own RFC 5545 datetime formatting, and your own
  rate-limit handling — `gcalcli` does all of that and exposes
  agenda/add/search/import/remind as one-liners. Drop to raw
  API only when you need a Calendar feature `gcalcli` has not
  surfaced yet (e.g. ACL changes on a shared calendar).

## Caveats

- **OAuth setup is a one-time chore.** You need to create a
  Google Cloud project, enable the Calendar API, create an OAuth
  client ID (Desktop app), and feed those credentials into
  `gcalcli init`. The README walks through it; budget 10 minutes
  the first time.
- **Token lives at `~/.gcalcli_oauth` in plaintext.** Treat the
  file like an SSH key: 0600 perms, no syncing it to a public
  dotfiles repo. Revoke from <https://myaccount.google.com>
  → Security → Third-party apps if a machine is lost.
- **`quick` parsing is server-side and English-biased.** The
  natural-language event parser is Google's, not gcalcli's, and
  it is best at en-US date/time idioms. For non-English locales
  or unambiguous machine input, prefer `gcalcli add --when …
  --duration …`.
- **Recurring events are first-class for read, awkward for
  write.** Editing one instance of a recurring series ("move
  *just* next Tuesday's standup") is supported but verbose; for
  heavy recurring-event surgery the Calendar web UI is still
  faster.
- **No write access to other people's calendars without sharing.**
  Standard Calendar permissions apply; `gcalcli` cannot bypass
  them, and trying to `add` to a calendar you only have
  free/busy access to will return a 403.
