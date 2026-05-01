# smassh

- **Repo:** https://github.com/Kraanzu/smassh
- **Version:** v2.5.0 (latest tagged release line known at cataloging — verify with `pip show smassh` after install if a newer 2.x is out)
- **License:** MIT ([LICENSE](https://github.com/Kraanzu/smassh/blob/main/LICENSE))
- **Language:** Python (Textual TUI)
- **Install:** `pip install smassh` · `pipx install smassh` · binary name is `smassh`

## What it does

`smassh` is a terminal typing-test app built on Textual. It runs
a monkeytype-style speed and accuracy drill entirely inside the
terminal: a target string is rendered, you type, characters
recolour green/red as you go, WPM and accuracy update live,
and a result screen at the end shows raw WPM, net WPM,
accuracy, consistency, and the per-keystroke time series. The
test corpus, length, language, punctuation/numbers toggles,
theme, cursor style, blind mode, sound feedback, and key bindings
all live in a TOML config and can be flipped from the in-app
settings screen without restarting. Multiple word lists ship in
the box (English in several frequency tiers, several other
languages, code-symbol corpora) and custom corpora are a single
text file. Results persist locally as JSONL so you can plot a
WPM trend over weeks without sending anything to a server.

## When to pick it / when not to

Reach for `smassh` when you want a **local, offline, ad-free
typing trainer** that lives where your editor lives. It's the
right pick when network typing sites (monkeytype, keybr,
typeracer) are blocked, when you want your practice data to
stay on your machine, when you live in `tmux` and never want
to leave the terminal, or when you need a typing drill against
a custom corpus (your own codebase identifiers, an API surface
you keep mistyping). The Textual UI is genuinely pleasant —
animations, themes, sane defaults — and the config file makes it
easy to commit a "team typing standard" alongside dotfiles.

Skip it if you want competitive multiplayer (`typeracer` and
the web typing sites own that niche), if you need the very
specific metrics one of those sites provides (typeracer-style
ELO, monkeytype's exact theme catalogue), or if your terminal
emulator does not handle Textual's full-screen rendering well
(modest legacy terminals over slow SSH may flicker — measure
before relying on it). It is also Python-stack — fine for a
typing app, but not zero-deps; if a no-Python policy is firm,
fall back to one of the Go/Rust typing CLIs.

## Why it matters in an AI-native workflow

Agent loops have made the keyboard latency floor *more* of a
bottleneck, not less: the human is still the one composing the
prompt, accepting the diff, and writing the review comment.
Five extra WPM and one fewer backspace per sentence compounds
across a working day spent in `git`/`gh`/agent CLIs. A
local-first typing tool with a custom corpus lets you drill
against the exact identifiers and shell commands you actually
type — `kubectl`, `terraform`, your own service names — rather
than against a generic English word list. And because results
stay local, the "did my typing improve this month" question
has an answer that does not require a SaaS account.

## Example invocations

```bash
# Launch the TUI with default settings
smassh

# In-app: ctrl+s opens settings, ctrl+t toggles theme,
# tab + enter restart the test, esc exits.

# Point at a custom word list (one word per line)
smassh --words ~/dotfiles/typing/k8s-vocab.txt

# Reset stored results and start fresh
rm ~/.local/share/smassh/results.jsonl

# Plot WPM over time with any JSONL-aware tool, e.g. jq + gnuplot
jq -r '[.timestamp, .wpm] | @tsv' \
  ~/.local/share/smassh/results.jsonl | \
  gnuplot -p -e 'plot "<cat" using 1:2 with lines'
```

## Alternatives in this catalog

- [`vhs`](../vhs/) — record terminal sessions as gifs; useful for
  capturing a typing demo, not for measuring it.
- [`atuin`](../atuin/) — shell history sync; orthogonal, but the
  same "your keystrokes are your data, keep them local" stance.
- [`asciinema`](../asciinema/) — record-and-share terminal casts;
  pairs with smassh if you want to publish a typing demo.
