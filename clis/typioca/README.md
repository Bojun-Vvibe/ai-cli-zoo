# typioca

> **Cosy terminal typing-test TUI** with time-bounded,
> word-bounded, sentence-bounded, and custom-corpus modes —
> renders WPM / accuracy / consistency over a live chart and
> stores per-run history locally for trend tracking. Pinned to
> **v3.1.0**
> ([LICENSE](https://github.com/bloznelis/typioca/blob/master/LICENSE),
> MIT).

Source: <https://github.com/bloznelis/typioca>

## TL;DR

`typioca` is a Bubble Tea / Lipgloss TUI typing trainer for
people who live in the terminal and don't want to context-
switch to `monkeytype.com` to practice. Pick a mode (time:
15/30/60/120 s, words: 10/25/50/100, sentences: a configurable
count of full natural sentences, or custom: feed it any text
file as the source corpus), pick a word-list (English with
common-1k / common-3k / 10k variants ship in-binary, plus
punctuation + numbers toggles), and start typing. The
top-line shows the target text with the current cursor
position colored, mistyped characters highlight in red and
*stay* highlighted (no auto-correct, no auto-skip), and a
live status row updates WPM / accuracy / time-remaining each
keystroke. On finish you get a results screen with a
`go-echarts`-rendered ASCII line plot of WPM-over-time +
errors-over-time, plus a "Show stats" view that walks back
through your last N runs from `~/.local/share/typioca/`
(SQLite) so you can see whether last week's punctuation
practice actually moved your accuracy. The whole thing is a
single static Go binary, ~7 MB, no runtime deps, runs over
SSH on a 80x24 TTY.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install typioca

# Go install (any platform with Go 1.22+)
go install github.com/bloznelis/typioca@v3.1.0

# Single-binary download (GitHub releases)
curl -L -o typioca.tar.gz \
  https://github.com/bloznelis/typioca/releases/download/3.1.0/typioca_3.1.0_linux_amd64.tar.gz
tar xzf typioca.tar.gz && sudo mv typioca /usr/local/bin/

# macOS arm64
curl -L -o typioca.tar.gz \
  https://github.com/bloznelis/typioca/releases/download/3.1.0/typioca_3.1.0_darwin_arm64.tar.gz
tar xzf typioca.tar.gz && sudo mv typioca /usr/local/bin/

# Build from source pinned to tag
git clone --depth 1 --branch 3.1.0 https://github.com/bloznelis/typioca.git
cd typioca && go build -o typioca .
sudo mv typioca /usr/local/bin/
```

## Usage

```bash
# Just launch — picker UI walks you through mode + corpus
typioca

# Skip the picker: 30-second time test, English common-1k, with punctuation
typioca --mode time --duration 30s --words common-1k --punctuation

# Word-count mode, 50 words, with numbers mixed in
typioca --mode words --count 50 --words common-3k --numbers

# Sentence mode, 5 natural-language sentences (capitalization + punctuation built in)
typioca --mode sentences --count 5

# Custom corpus — feed it any text file
typioca --mode custom --source ~/notes/python-snippets.txt

# Re-open the historical stats view without taking a test
typioca stats
```

Inside a test: `Tab` restarts with the same settings, `Esc`
returns to the menu, `Ctrl-C` quits. After a test: `r` repeats,
`s` shows historical stats, `n` picks a new mode.

## Why it's interesting

The terminal-typing-test slot has three real entries —
[`tt`](https://github.com/lemnos/tt) (single-screen Go binary,
zero history, opinionated minimal UI),
[`thokr`](https://github.com/coloradocolby/thokr) (Rust, slick
real-time chart but no persistent history and the project is
inactive since 2022), and
[`smassh`](https://github.com/kraanzu/smassh) (Textual / Python,
heaviest deps but the most config knobs) — and `typioca` sits
in the middle on every axis: heavier than `tt` (you get a
results chart and a real history database), lighter than
`smassh` (one Go binary, no Python runtime, no theming
config file required to look good). The differentiator is
**sentence mode**: `tt` and `thokr` only do word streams,
which is great for raw WPM but bad for practicing the actual
shape of typing (capitalization at sentence start, period +
space + capital, comma cadence, parenthetical asides). `typioca`
ships a curated set of full sentences and the custom-corpus
hook lets you point it at your own writing — paste a chapter
of your draft into a `.txt` and practice the exact prose
you'll be writing tomorrow. The persistent stats DB matters
when you actually want to track improvement: per-run rows
keyed by mode + word-list + duration mean "did punctuation
practice help my non-punctuation accuracy" is a real query
instead of a guess. Pick `typioca` when (a) you want
sentence-shape practice not just word streams, (b) you want
local trend tracking without signing up for a SaaS, or (c)
you SSH into dev boxes and want a typing test that runs
where you already are. Skip it for raw competitive WPM
optimization (use `tt` for the fastest single-screen
benchmark) or if you need code-mode (typing real source code
with brackets / indentation enforced) — neither `typioca`
nor any of its peers do that well; `monkeytype` web is still
the answer there. Active maintenance through 2025, releases
cut by GitHub Actions on tag push, semver since 3.0.
