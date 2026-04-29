# toipe

> **A terminal typing-speed tester written in Rust** — pulls a word
> list, draws a single-screen prompt, scores you on WPM / accuracy /
> per-keystroke latency, and prints a ranked summary on exit. Pinned
> to **v0.5.0**
> ([LICENSE](https://github.com/Samyak2/toipe/blob/main/LICENSE),
> MIT).

Source: <https://github.com/Samyak2/toipe>

## TL;DR

`toipe` is a tiny single-binary typing tester that runs entirely in
the terminal — no browser, no account, no leaderboard. Launch
`toipe`, it shows ~30 random English words, you type them, it scores
**WPM** (gross + net), **accuracy** (correct chars / total chars),
**raw vs corrected** speed, and per-keystroke timing. Word lists are
selectable: `top1000` (default), `top250`, `top10000`, `os` (local
`/usr/share/dict/words`), or any text file you point at with `-f`.
Designed for the "warm up before a long coding session" or "five
minutes of typing practice that does not need an internet
connection" loop. After each round it prints a coloured summary and
asks `[r]etry / [n]ew / [q]uit` — the entire interaction stays on
the keyboard.

## Install

```bash
# Cargo (the canonical install — no Homebrew formula yet)
cargo install --locked toipe

# Arch Linux (AUR)
# yay -S toipe

# from a release binary
curl -L -o toipe \
  https://github.com/Samyak2/toipe/releases/download/v0.5.0/toipe-x86_64-unknown-linux-gnu
chmod +x toipe && sudo mv toipe /usr/local/bin/

# verify
toipe --version    # toipe 0.5.0
```

Requires a UTF-8 capable terminal with at least 80×24 cells; works
inside tmux and screen.

## License

MIT — see
[LICENSE](https://github.com/Samyak2/toipe/blob/main/LICENSE).
Permissive, redistributable, no attribution required for binaries.

## One Concrete Example

```bash
# 1. default: 30 words from the top-1000 English list
toipe

# 2. shorter or longer rounds
toipe -w 15      # 15 words (~30 s for a 60 WPM typist)
toipe -w 100     # 100 words (~2 min)

# 3. switch the word source
toipe -l top250        # only the most frequent 250 words
toipe -l top10000      # broader vocabulary, includes rarer words
toipe -l os            # uses /usr/share/dict/words

# 4. type a custom file (great for practising language-specific tokens)
toipe -f rust-keywords.txt
toipe -f my-prompts.txt        # practise typing your most-used LLM prompts

# 5. include punctuation (default lists are lowercase no-punct)
toipe -p

# 6. typical session
toipe -w 50 -p
# > the quick brown fox, jumps over the lazy dog. ...
# (you type)
# WPM: 78  Accuracy: 96.4%  Time: 38.4s  Mistakes: 7
# [r] retry same prompt   [n] new prompt   [q] quit
```

## Niche It Fills

**The offline, in-terminal typing drill.** Web typing testers
(monkeytype.com, 10fastfingers, keybr) need a browser, an account
for stats, and tend toward gamification. `toipe` fits the
"coding-shell muscle memory" use case: warm up before a long session,
practise the keys you keep mistyping, drill a custom word list of
shell built-ins / Rust keywords / Vim commands / LLM prompt fragments
— all without leaving the terminal you are about to work in.

## Why use it

1. **Truly offline + zero-config.** Single Rust binary, no network
   call, no config file, no account; first run works on a fresh
   machine in 30 seconds (`cargo install`).
2. **Bring-your-own word list (`-f file.txt`).** Practise the actual
   tokens you type all day — your team's domain vocabulary, the
   prompts you keep retyping into [`mods`](../mods/) /
   [`aichat`](../aichat/), the Vim normal-mode commands you fumble.
3. **Honest metric reporting.** Reports gross WPM, net WPM, raw
   accuracy, *and* mistakes — no single-number gamified score that
   hides whether you slowed down to type accurately or sped up by
   mashing.

## Vs Already Cataloged

- **Nothing else in the catalog overlaps directly.** `toipe`'s
  niche — terminal typing practice — is unique in this catalog.
  The closest neighbours conceptually are utilities like
  [`hyperfine`](../hyperfine/) (benchmark *commands*, not human
  typing) and [`tealdeer`](../tealdeer/) (read short docs in the
  terminal); neither tests typing speed.

## Caveats

- **English-only word lists by default.** The bundled lists are
  English; for other languages use `-f <path>` with your own word
  list. There is no built-in Unicode-aware tokenizer for CJK input
  methods — `toipe` measures Latin-script keystrokes, not IME
  composition.
- **Single-binary, no history.** `toipe` does not persist past
  scores anywhere; if you want to track WPM over time, redirect
  the summary line to a log file yourself
  (`toipe 2>&1 | tee -a ~/typing.log`) and grep / chart later.
- **Terminal-bound input fidelity.** `toipe` measures what your
  terminal hands the program — modifier keys (Shift / AltGr) and
  IME compositions are not separately scored, and a flaky SSH /
  tmux setup with high input latency will distort per-keystroke
  timing.
- **No competitive multiplayer.** Single-player only by design;
  for racing against others, the browser testers (monkeytype) are
  the right tool.
- **Project pace is slow.** Last release v0.5.0; treat it as a
  stable small tool rather than a moving target. The MIT licence
  and small Rust codebase make local forks straightforward if you
  need a feature.
