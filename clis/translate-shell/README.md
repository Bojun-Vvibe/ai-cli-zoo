# translate-shell

> **Command-line interface to free online translation
> engines** (Google Translate, Bing Translator, Yandex,
> Apertium) — a single self-contained `gawk` script that
> exposes language detection, source/target language pairs,
> phonetic transliteration, dictionary lookup, batch file
> translation, an interactive REPL (`-I`), terminal speech
> synthesis (`-p`), and an emacs.el helper, all reachable as
> `trans en:fr "hello world"` with no API key, no SDK, and no
> daemon. Pinned to **v0.9.7.1**, SPDX `Unlicense`,
> [LICENSE](https://github.com/soimort/translate-shell/blob/develop/LICENSE.txt).

Source: <https://github.com/soimort/translate-shell>

## TL;DR

`translate-shell` (binary name: `trans`) is a ~5000-line `gawk`
program that pretends to be a browser, hits the public
endpoints of Google / Bing / Yandex / Apertium, parses the
response, and prints a colourised, terminal-friendly
translation card: source phrase, detected language, target
translation, phonetic IPA, alternate meanings, part-of-speech
breakdown, and example sentences. No account, no API token,
no sign-up — it works the moment you have `gawk` on PATH.
Two operating modes: **one-shot** (`trans :ja "Hello, world."`
prints the card and exits) and **interactive** (`trans -I`
opens a REPL where each line is translated in place, useful
for chat conversations and language study). Useful adjuncts:
`-d` for dictionary-style lookup, `-b` for terse "just the
translation" mode (suitable for piping into `sed` / `xargs`),
`-p` to read the result aloud via the platform speech
synth, `-download-audio` to grab the MP3, and `-shell` to
enter an emacs-style editing loop.

## Install

```bash
# Homebrew
brew install translate-shell

# Arch
sudo pacman -S translate-shell

# Debian / Ubuntu
sudo apt install translate-shell

# From source (single-file install)
wget git.io/trans
chmod +x trans
sudo mv trans /usr/local/bin/

# verify
trans -V    # Translate Shell 0.9.7.1
```

Requires `gawk` (not plain `awk`). For TTS: `mpv` or `mplayer`
on Linux, `say` is built-in on macOS.

## License

The Unlicense (public domain dedication) — see
[LICENSE.txt](https://github.com/soimort/translate-shell/blob/develop/LICENSE.txt).
No restrictions on use, redistribution, or modification.

## Representative Commands

```bash
# 1. one-shot translation, auto-detect source
trans "Wie geht es dir?"

# 2. explicit source:target, multi-line input via stdin
echo "this is a test" | trans en:zh-CN

# 3. brief mode for shell pipelines
SUBJ=$(trans -b en:de "weekly status")
git commit -m "$SUBJ"

# 4. dictionary lookup (synonyms, examples, part of speech)
trans -d en: "ephemeral"

# 5. translate a whole file
trans -i article.en.md -o article.de.md en:de

# 6. interactive REPL — translate line-by-line for a chat convo
trans -I :es

# 7. swap engines (google is default; bing has different LLM tone)
trans -e bing :fr "deploy the agent to production"

# 8. speak the translation aloud (macOS uses `say`)
trans -speak :ja "good morning"
```

## Niche / Category

Network-translation client / language utility (zero-key,
zero-SDK).

## Why It Is Orthogonal

The catalogue's existing language and text utilities solve
adjacent but different problems: [`harper`](../harper/) is an
**English-only grammar / style linter**, [`vale`](../vale/) is
an **English prose-style checker**, [`tealdeer`](../tealdeer/)
and [`tldr`](../tldr/) translate *commands*, not *human
languages*. `translate-shell` is the only entry that turns a
shell into a polyglot translator without requiring an API key,
SDK, or browser session. Pairs with [`mblaze`](../mblaze/) /
[`himalaya`](../himalaya/) (translate inbound non-English
mail in a pipe — `mshow $msg | trans -b :en`), [`glow`](../glow/)
or [`mdcat`](../mdcat/) (preview translated Markdown side by
side), [`watchexec`](../watchexec/) (auto-translate a `.po`
file on save during i18n work), and [`yt-dlp`](../yt-dlp/)
(`yt-dlp --get-title URL | trans :en` to title-translate a
foreign-language video before deciding to download). Reach
for it when the workflow is *terminal-native and ad-hoc*: a
quick "what does this Stack Overflow Russian comment mean?",
a chat-window REPL during a bilingual conversation, a CI
step that auto-translates release notes, or a one-liner that
captions screenshots before posting. Caveats: the public
endpoints are rate-limited and undocumented (occasional
throttling, breaking parser changes), so it is *not* the
right tool for high-volume production translation — for
that, point the workflow at a paid API. For one-off,
keyless, shell-shaped translation it has no peer in this
catalogue.
