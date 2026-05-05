# gtt

> **Translate text across eight engines from a TUI** — a single
> Go binary that wraps Apertium, Bing, ChatGPT, DeepL, DeepLX,
> Google, LibreTranslate, and Reverso behind one keyboard-driven
> screen. Pinned to **v11** (commit
> `ff4e42927ea4b656c1a82b558f8479282d974dac`,
> [LICENSE](https://github.com/eeeXun/gtt/blob/master/LICENSE),
> MIT).

Source: <https://github.com/eeeXun/gtt>

## TL;DR

`gtt` ("Google Translate TUI" originally; now polyglot across
backends) is a `tview`-based terminal application that opens two
text panes (source on the left, target on the right), a status
bar, and a one-key cycle for translator engine, source language,
and target language. Type or paste in the source pane, press
`<Enter>` (or wait for debounce), and the translation lands in
the target pane via whichever backend is currently selected.
Translation history (`Ctrl-h`), pronunciation playback for
languages the engine supports (`Ctrl-p`, requires `mplayer` or
`mpv`), one-shot stdin mode for piping (`echo "bonjour" | gtt
-x`), and a JSON output mode (`-j`) for scripting are all reachable
from one ~12 MB binary.

## Install

```bash
# Go install (any platform)
go install github.com/eeeXun/gtt@latest

# Homebrew (macOS / Linux)
brew install gtt

# Arch
yay -S gtt-bin

# from a release tarball
curl -Lo gtt.tar.gz "https://github.com/eeeXun/gtt/releases/download/v11/gtt_Linux_x86_64.tar.gz"
tar xf gtt.tar.gz
sudo install gtt /usr/local/bin/

# verify
gtt --version    # 11
```

Configuration lives in `~/.config/gtt/config.toml` (created on
first launch). API-keyed engines (DeepL, ChatGPT, Bing) need
keys in that file or in env (`DEEPL_KEY`, `OPENAI_API_KEY`,
`AZURE_TRANSLATOR_KEY`). Apertium, Google, LibreTranslate, and
Reverso are key-less out of the box.

## License

MIT — see
[LICENSE](https://github.com/eeeXun/gtt/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. open the TUI on the default engine (Google) and language pair
gtt

# 2. one-shot translate a string to French and print to stdout
gtt -x -s en -d fr "the quick brown fox jumps over the lazy dog"

# 3. read from stdin, write JSON for a script
echo "guten Morgen" | gtt -x -s de -d en -j
# {"src_lang":"de","tar_lang":"en","src_text":"guten Morgen",...

# 4. pick a different backend (DeepL) for one invocation
gtt -x -t deepl -s en -d ja "production deployment was successful"

# 5. translate a file paragraph-by-paragraph with xargs
cat notes.txt | xargs -d '\n' -I{} gtt -x -s en -d es "{}"

# 6. inside the TUI:
#    Ctrl-j / Ctrl-k     cycle source / target language
#    Ctrl-t              cycle translator backend
#    Ctrl-s              swap source <-> target
#    Ctrl-p              play TTS pronunciation of source pane
#    Ctrl-h              open history pane
#    Ctrl-q              quit
```

## Niche It Fills

**Translation as a uniform terminal verb across competing engines.**
The default workflow for "translate this paragraph" is opening a
browser tab, pasting into translate.google.com / deepl.com, and
copying the result back. `gtt` collapses that loop into a tmux
pane that already has the language pair and the engine you prefer
selected, plus the secondary value of trivially A/B-ing two
engines on the same input (Google's gloss vs DeepL's idiom-aware
rewrite vs ChatGPT's paragraph-aware reflow), without juggling
three browser tabs and three account states.

## Why use it

1. **Eight backends behind one schema.** Switching between
   Apertium (open-data), Google (broad coverage), DeepL (best for
   en↔de/fr/ja), ChatGPT (preserves context across paragraphs),
   and LibreTranslate (self-hostable) is one keystroke (`Ctrl-t`)
   inside the TUI or one flag (`-t deepl`) in one-shot mode.
   Same input pane, same output pane, same history. The cost of
   evaluating "which engine handles this domain best" drops to
   pressing one key three times.
2. **Stdin-aware pipeline mode.** `gtt -x` reads from stdin or
   argv and writes plain text (or JSON with `-j`) to stdout, so
   wrapping translation into a `Makefile`, a CI doc-build, or a
   shell function is trivial. The TUI is the default but not the
   only entry point.
3. **No phone-home, no account required for half the engines.**
   Apertium, Google (free public endpoint), LibreTranslate, and
   Reverso work without API keys; the keyed engines (DeepL,
   ChatGPT, Bing) read from env or config and never bake creds
   into the binary. Self-hosted LibreTranslate at
   `https://lt.example.org` is a one-line config change.

For agent CLIs that need a translation step (translate a user
prompt to English before the model call, translate the model's
output back), `gtt -x -j` is a clean tool-call shape that returns
structured JSON suitable for parsing in a tool wrapper.

## Vs Already Cataloged

- **Vs [`translate-shell`](../translate-shell/):** the closest
  peer. Both wrap multiple translation backends behind one CLI.
  `translate-shell` (`trans`) is older, AWK-based, and broader
  on language pair detection heuristics; `gtt` is Go-binary,
  TUI-first, and ships interactive engine swap + history pane out
  of the box. Pick `trans` for shell-script-first workflows where
  you never want a TUI; pick `gtt` when the primary mode is "open
  the translator, work in it for a while, switch engines to
  compare". They coexist fine.
- **Vs [`smartcat`](../smartcat/):** orthogonal — `smartcat`
  is an LLM-first CLI that uses a chat model as a transformer
  for text in a pipe (translation is one of many prompts);
  `gtt` is a translation-specific frontend with translation-shaped
  UI (language pair selector, history of `(src_lang, tar_lang,
  src, tar)` tuples). Pick `smartcat` when the verb is
  "rewrite this text in the style of X"; pick `gtt` when the
  verb is "translate this text from A to B".
- **Vs `curl translate.googleapis.com/...`:** `gtt` packages
  the same free Google endpoint plus seven others behind a
  TUI + flag schema, with debounce, history, and TTS — replaces
  a directory of one-off translation shell scripts with one
  configurable binary.

## Caveats

- **Free Google endpoint is unofficial and rate-limited.** Heavy
  use (hundreds of calls/minute) will get IP-throttled. For
  production batch translation, configure DeepL or self-hosted
  LibreTranslate.
- **TTS playback requires `mplayer` or `mpv` in `PATH`.** The
  TUI omits the play-pronunciation key when neither is present;
  no error, just a missing affordance.
- **API keys are read from env or `config.toml`, in plaintext.**
  No keychain integration. On a shared box, set the env var
  per-shell rather than committing the config file.
- **Engine selector state is persisted across runs.** If you
  forget you switched to DeepL three days ago, the next launch
  still uses DeepL. The bottom-bar shows the current engine —
  glance at it before pasting.
- **Language detection is engine-dependent.** Apertium and
  LibreTranslate require explicit `-s` / `-d`; Google and DeepL
  auto-detect. Mixed-language input (English with Japanese
  loanwords) is interpreted by whichever engine you chose.
