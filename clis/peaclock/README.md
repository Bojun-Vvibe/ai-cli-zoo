# peaclock

> **A responsive and customizable clock, timer, and stopwatch
> for the terminal** — a single C++17 binary that fills your
> entire terminal with an ASCII, digital, or binary clock face,
> resizes responsively, supports themes, locales, timezones,
> and a built-in command prompt for live reconfiguration. Pinned
> to **v0.4.3**
> ([LICENSE](https://github.com/octobanana/peaclock/blob/master/LICENSE),
> MIT).

Source: <https://github.com/octobanana/peaclock>

## TL;DR

`peaclock` is the terminal clock you put up on a spare monitor
when you want to glance at the time without a desktop widget.
Three modes (`clock`, `timer`, `stopwatch`) × three views
(`ascii`, `digital`, `binary`) cover most needs. The clock is
auto-sized: it grows to fill the terminal and re-flows on
resize, optionally constrained to an aspect ratio so digits
don't get stretched. Colors are 4-bit, 8-bit, or 24-bit; you can
toggle a "party mode" that cycles colors, attach a custom date
string, switch locales/timezones at runtime, or hand it a shell
command via `timer-exec` to run when the timer hits zero (handy
for Pomodoro: `timer-exec "say break time"`). Configuration is
plain-text, hot-reloaded from `~/.peaclock/config`, and a
fuzzy-searchable command history lets you experiment live.

## Install

```bash
# Homebrew (macOS / Linux) — community formula
brew install peaclock

# Build from source (any Unix)
git clone --depth 1 --branch 0.4.3 \
  https://github.com/octobanana/peaclock.git
cd peaclock
./RUNME.sh build           # CMake >= 3.8, GCC >= 8 or Clang >= 7, ICU >= 62.1
./RUNME.sh install         # installs into /usr/local/bin

# macOS note: Apple Clang is too old for std::filesystem.
# Install Homebrew GCC first:
brew install gcc icu4c
./RUNME.sh build -- -DCMAKE_CXX_COMPILER="$(brew --prefix gcc)/bin/g++-14"

# Verify
peaclock --version    # peaclock 0.4.3
```

## Example usage

```bash
# 1. Plain clock, fills the terminal, default ASCII view
peaclock

# 2. 25-minute Pomodoro timer that says "break" when done
peaclock --mode timer --timer 25:00 \
         --timer-exec 'say "break time"'

# 3. Stopwatch in digital view, 24-hour format, no seconds
peaclock --mode stopwatch --view digital --hour-24 --no-seconds

# 4. Binary clock view with unicode block chars (cfg/binary-unicode)
peaclock --config ./cfg/binary-unicode

# 5. Pin a specific timezone and locale for an away-team display
peaclock --timezone Asia/Tokyo --locale ja_JP.UTF-8 \
         --date '%A %Y-%m-%d'

# 6. Use a custom config dir (XDG-friendly)
mkdir -p ~/.config/peaclock
cp ./cfg/default ~/.config/peaclock/config
alias peaclock='peaclock --config-dir ~/.config/peaclock'

# 7. Live reconfigure: press ':' inside peaclock to open the
#    command prompt, then for example:
#      :color 33
#      :view binary
#      :date %H:%M  %Z
#      :mkconfig! ~/.peaclock/config   # save current state

# 8. Common keybindings while running:
#    q        quit
#    space    toggle pause (timer/stopwatch)
#    r        reset (timer/stopwatch)
#    j / k    swap mode / view
#    ?        in-prompt help
```

## Why this lives in the zoo

Most "TUI clock" tools are either fixed-size ASCII art (`tty-clock`,
`peaclock`'s ancestors) or full-blown TUI dashboards (`wtfutil`).
`peaclock` is the rare tool that does only one thing — display
time — but does it with the polish of a desktop widget: real
responsive layout, real timezone/locale handling, real
truecolor, real timer-exec hooks. The runtime command prompt
plus `mkconfig!` makes it unusually pleasant to iterate on a
look without restarting, which is what makes it survive on the
spare-monitor display long after lesser clocks get killed.
