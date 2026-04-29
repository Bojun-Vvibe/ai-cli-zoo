# asciinema

> **A terminal session recorder that captures keystrokes and output
> as a small text-based JSON cast file (not a video) and replays it
> losslessly in any terminal or in a browser via an embeddable
> player** — copy-paste works mid-playback, the recordings are
> diffable in git, and a 30-minute session is typically under
> 200 KB. Pinned to **v3.2.0** (commit
> `202d5c5761687b489451e9bb1a5fe9189b73e9d9`,
> [LICENSE](https://github.com/asciinema/asciinema/blob/develop/LICENSE),
> GPL-3.0).

Source: <https://github.com/asciinema/asciinema>

## TL;DR

`asciinema` is what you reach for when you need to demo a CLI
workflow in a README, a bug report, or a tutorial and reaching for
a screen recorder feels wrong because the result is a 40 MB MP4
that pixelates on retina displays and where viewers cannot copy
the commands they just watched. `asciinema rec demo.cast` drops
into a recording subshell — type as normal, exit with `Ctrl-D` —
and writes a JSON Lines file containing the timed stream of
terminal output. `asciinema play demo.cast` replays it in your
terminal at original speed (or `-s 2` for 2×, or `-i 1` to cap
idle time at 1 s). Upload with `asciinema upload demo.cast` for a
shareable URL on asciinema.org, or self-host the JS player and
embed in your own docs site. The 3.x rewrite is a single Rust
binary (the 2.x line was Python) — faster startup, no
`pip install`, drops on any box with `cargo install` or a
prebuilt release tarball.

## Install

```bash
# Homebrew (macOS / Linux)
brew install asciinema

# Cargo (any platform with a Rust toolchain)
cargo install --locked asciinema

# Linux package managers
# Debian/Ubuntu: apt install asciinema   (may still be 2.x)
# Arch:          pacman -S asciinema
# Fedora:        dnf install asciinema
# Nix:           nix-env -iA nixpkgs.asciinema

# from a release tarball (any OS) — see
# https://github.com/asciinema/asciinema/releases

# verify
asciinema --version    # asciinema 3.2.0
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/asciinema/asciinema/blob/develop/LICENSE).
The recorder binary is freely redistributable; GPL-3.0 terms apply
if you ship a derivative built from asciinema sources. The cast
files you produce are your own — no license is imposed on them.

## One Concrete Example

```bash
# 1. record a session — drops you into a subshell, exit with Ctrl-D
asciinema rec demo.cast

# 2. record with a max idle gap so 60-second pauses become 2 seconds
asciinema rec --idle-time-limit 2 demo.cast

# 3. record with a title and command-only run (no interactive subshell)
asciinema rec --title "fzf demo" --command "fzf --height 40%" demo.cast

# 4. play back at 2× speed, capped at 1-second idle gaps
asciinema play -s 2 -i 1 demo.cast

# 5. inspect the cast — it is JSON Lines, header on line 1, then
#    [timestamp, "o", "output bytes"] tuples
head -3 demo.cast
# {"version": 2, "width": 120, "height": 30, "timestamp": 1714400000, ...}
# [0.123, "o", "$ "]
# [0.456, "o", "ls\r\n"]

# 6. upload to asciinema.org for a shareable URL
asciinema upload demo.cast
# https://asciinema.org/a/abc123

# 7. self-host the embeddable JS player in your docs site
#    (player JS is at https://github.com/asciinema/asciinema-player)
```

## Niche It Fills

**Lossless terminal-session capture as a small text artifact, not
a video.** The space is screen recorders (OBS, QuickTime, Loom,
ttyrec, terminalizer, `vhs`, `t-rec`). MP4 recorders give you a
heavy binary blob viewers cannot interact with. `vhs` (Charm) goes
the other direction — render a `.tape` script to an animated GIF
or MP4, no human in the loop. `terminalizer` is similar to
asciinema but produces GIF/MP4 by default and is npm-bound.
asciinema sits in the corner labelled *"capture a real human
session, store it as diffable text, replay losslessly, link to it
or embed an interactive player."* Pick asciinema when the viewer
should be able to *copy* the command they just watched, when you
need the cast file to live in a git repo next to a tutorial, or
when you want a 200 KB artifact instead of a 40 MB video.

## Why use it

Three things `asciinema` does that pay off immediately:

1. **The cast file is text.** Diffable in git, greppable for the
   exact command you ran, editable if you fat-fingered a password
   into the recording. A 30-minute session is typically under
   200 KB compared to tens of MB for an MP4 of the same content.
2. **Replay is lossless and interactive.** The viewer can pause,
   scroll back, and **copy text out of the playback** — including
   from the embedded JS player on a webpage. MP4 / GIF screen
   recordings cannot do this.
3. **Single binary, no driver.** No screen-capture permission, no
   audio device, no GPU encoder — `asciinema` reads from a PTY and
   writes JSON. Works the same on macOS, Linux, in a Docker
   container, over SSH, in a CI runner.

## Weakness

**It only captures terminal output.** If your demo involves a
browser window, a desktop GUI, mouse clicks outside the terminal,
or audio narration, asciinema cannot help — reach for `obs` or
QuickTime. Also, recordings of TUIs that abuse cursor positioning
(some full-screen apps with esoteric escape sequences) can replay
slightly off in older players; the v3 player handles the common
cases well but esoteric edge cases exist.

## When to choose

- You are writing a tutorial / bug report / README and want the
  reader to copy the commands you ran.
- You want the demo artifact to live in `git` next to the docs.
- You need to share a CLI session over chat / email / a PR comment
  with a sub-megabyte file or an embeddable URL.

## When NOT to choose

- The demo includes a browser, GUI app, or mouse — use a screen
  recorder.
- You want the *output*, not the *interaction* — generate it with
  [`vhs`](../vhs/) from a `.tape` script (reproducible, scriptable,
  no human required) and render to GIF / MP4 / webm.
- You need audio narration — reach for OBS / Loom / QuickTime.
