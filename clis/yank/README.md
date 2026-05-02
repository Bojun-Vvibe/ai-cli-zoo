# yank

> **Yank terminal output to the clipboard, picked with the keyboard.**
> A tiny C utility: pipe any text into `yank`, it tokenizes the
> stream, lets you pick one token with vi-keys, and copies it to
> the system clipboard. No mouse, no `tmux` selection mode, no
> "highlight with the trackpad and pray".
> Pinned to **v1.3.0**
> ([LICENSE](https://github.com/mptre/yank/blob/master/LICENSE),
> MIT).

Source: <https://github.com/mptre/yank>

## TL;DR

`yank` reads stdin, splits it into tokens by a configurable
delimiter regex (default: whitespace + common punctuation), and
shows the underlying text on stdout with the current token
highlighted. You move the highlight with `h` / `l` / arrow keys,
press Enter to copy that token to the clipboard, or `q` to abort.
It shells out to `xsel` / `xclip` (Linux), `pbcopy` (macOS), or
any command you set in `YANK_CLIPBOARD_CMD`. One C source file,
~600 lines, no runtime deps beyond a clipboard helper.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install yank

# Debian / Ubuntu
apt install yank

# Arch
pacman -S yank

# Alpine
apk add yank

# from source
git clone https://github.com/mptre/yank && cd yank
make && sudo make install PREFIX=/usr/local

# verify
yank -v          # 1.3.0
```

## License

MIT — see
[LICENSE](https://github.com/mptre/yank/blob/master/LICENSE).
Permissive, embed-friendly, no copyleft, no attribution beyond
preserving the notice. Safe to ship inside any internal tool or
distro image.

## One Concrete Example

```bash
# 1. yank an IP from `ip a` output
ip a | yank
# arrow-keys move between tokens; Enter copies the highlighted
# 192.0.2.17 to the clipboard, then yank exits silently.

# 2. yank a SHA from `git log`
git log --oneline -20 | yank -d ' '
# delimiter = single space, so each token is exactly one word;
# walk to the SHA, Enter, paste into your PR description.

# 3. yank a UUID-looking thing from a noisy log line
journalctl -u nginx --since "10 min ago" | yank \
    -- grep -oE '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}'
# trailing `-- cmd` mode: yank pipes the input through cmd first,
# tokens become whatever cmd emits per line.

# 4. custom clipboard helper (e.g. tmux buffer instead of system)
YANK_CLIPBOARD_CMD='tmux load-buffer -' \
    kubectl get pods | yank
# copies the chosen pod name into a tmux paste buffer.

# 5. only-once mode for scripts (first token, no UI)
echo "abc def ghi" | yank -1
# exits immediately after copying "abc"; useful in pipelines.
```

## Niche It Fills

**The "I see the value on screen, I want it in my paste buffer,
keyboard-only" gap.** Over SSH, in `tmux`, on a remote tty, or
just inside a terminal that has no working mouse selection
(Linux console, mosh, screen-readers), grabbing a single
token off the screen is annoying: tmux copy-mode requires
entering the mode, navigating word-wise, marking, yanking,
exiting; trackpad selection is mouse-dependent and grabs row
gutters / line numbers. `yank` is a one-shot pipe: command-pipe-
yank, walk to the token with `l`, Enter, done. It is the
spiritual ancestor of clipboard-helper TUIs and pairs naturally
with `fzf` for fuzzy-token selection on huge inputs.

## Why use it

Three things `yank` does that the obvious alternatives don't:

1. **Zero mode-switch.** No `prefix [` then navigate then `space`
   then navigate then `enter` then `prefix ]`. It's a unix filter
   that *exits* after one selection, so muscle memory is "command
   `| yank` Enter Enter".
2. **Tokenizer is a single regex flag.** `-d` accepts an extended
   regex that defines what a token *is*; for IPv6 / hashes /
   quoted strings you set the regex once and never fight word-at-
   a-time navigation.
3. **Clipboard backend is a string.** `YANK_CLIPBOARD_CMD` is just
   a shell command that reads stdin, so `yank` works identically
   over SSH (route through `xclip -selection clipboard` on the
   remote, or through OSC-52 with a one-line wrapper) without any
   tmux / X11 forwarding setup.

For an LLM-CLI workflow, `yank` is the **human-in-the-loop copy
step**: an agent prints "the failing pod is `api-7d8f-x9k2`",
the human runs `kubectl get pods | yank`, picks the pod, and
pastes it into the next prompt or a follow-up command without
touching the mouse. It's the inverse of `pbpaste | llm` — this
is `screen → clipboard` instead of `clipboard → llm`.

## Vs Already Cataloged

- **Vs [`fzf`](../fzf/):** `fzf` is a fuzzy *line* picker — you
  give it a list of options, it returns one line. `yank` picks a
  *token* from a single stream of text, where the stream wasn't
  designed as a list (it might be free-form log output). Pair
  them: `fzf` to pick a line, `yank` to pick a token from that
  line.
- **Vs tmux copy-mode:** tmux copy-mode is the entrenched answer
  inside tmux but useless outside it, modal, and slow for "just
  grab that one IP". `yank` is a unix filter that works in any
  terminal, including outside tmux.
- **Vs [`xclip`](../xclip/) / `pbcopy`:** Those copy *all of
  stdin* to the clipboard. `yank` copies *one token chosen
  interactively* from stdin. Different jobs: `cmd | xclip` when
  you want the whole output; `cmd | yank` when you want one
  field from it.
- **Vs OSC-52 escape sequences:** OSC-52 is the *transport* (how
  bytes get from the remote terminal into the local clipboard).
  `yank` is the *picker*. Compose them by pointing
  `YANK_CLIPBOARD_CMD` at an OSC-52 emitter for clipboard-over-
  SSH-without-X11.

## Caveats

- **Single token only.** No multi-select, no range select, no
  "copy these three fields joined by tab". For multi-field
  extraction reach for `awk` / `cut` and pipe the result to
  `xclip` / `pbcopy` directly.
- **Tokenizer is regex over bytes, not structure.** `yank` does
  not understand JSON / YAML / HTML; it sees the rendered text.
  For structured pickers use `jq` + `fzf` or `gron` + `fzf`.
- **Clipboard helper must exist.** On a headless box with no
  `xclip` / `xsel` / `pbcopy` and no OSC-52-aware terminal,
  `yank` has nowhere to put the bytes. Set
  `YANK_CLIPBOARD_CMD=cat` in that case to just print the
  selection to stdout.
- **Maintenance is light.** v1.3.0 (2022) is the latest tag;
  the project is feature-complete and the source tree is small,
  so bug-fix cadence is "as needed". Don't expect new features.
- **No mouse mode.** Deliberately. If you want mouse-driven
  terminal selection, your terminal emulator already does it;
  `yank` exists for the case where you *don't* want to use it.
