# moar

> **A friendlier `less(1)` pager** — written in Go by Johan
> Walles, with sensible defaults: search is case-insensitive
> until you type a capital, ANSI colours and 24-bit truecolour
> are passed through unmangled, line wrapping is on by default
> with `\` continuation marks, status line shows percent +
> filename + line count, `h` opens a built-in help, and `q`
> just quits without complaining about "no previous regular
> expression". Pinned to **v2.12.3** (commit
> `406c2746955638e77b890f2188236322782ec575`,
> [LICENSE](https://github.com/walles/moar/blob/master/LICENSE),
> BSD-2-Clause).

Source: <https://github.com/walles/moar>

## TL;DR

Most operators inherit `less` and never reconfigure it; the
defaults assume a 1990s VT100 (case-sensitive search, no colour
passthrough, mysterious key bindings, a `:` prompt for half the
commands). `moar` is a drop-in `PAGER` / `MANPAGER` replacement
that flips every default to what a 2020s terminal user actually
wants: smart-case search (lowercase = case-insensitive,
uppercase = exact), full ANSI + 24-bit colour passthrough so
`git diff`, `bat`, `delta`, `rg --color=always`, and `ls
--color` render correctly inside the pager, line wrapping on
by default with a visible continuation marker, single-key quit
(`q` always, no `:q`), forward search with `/` and reverse
with `?` — and a built-in `h` help screen so you do not have
to keep the `less(1)` man page open in another tab. One static
Go binary, no config file required, follows `$LESS` / `$MOAR`
env conventions.

## Install

```bash
# Homebrew (macOS / Linux)
brew install moar

# Go install
go install github.com/walles/moar@latest

# release tarball (any OS / arch)
curl -L https://github.com/walles/moar/releases/download/v2.12.3/moar-v2.12.3-darwin-arm64 \
  -o /usr/local/bin/moar && chmod +x /usr/local/bin/moar

# Arch
yay -S moar

# verify
moar --version    # moar 2.12.3
```

Wire it in as the system pager:

```bash
echo 'export PAGER=moar'                >> ~/.zshrc
echo 'export MANPAGER="moar -no-linenumbers"' >> ~/.zshrc
echo 'export MOAR="--no-statusbar --quit-if-one-screen"' >> ~/.zshrc   # optional tuning
```

`git`, `man`, `psql`, `kubectl`, `journalctl` all honour
`$PAGER` and start using `moar` after a shell restart.

## License

BSD-2-Clause — see
[LICENSE](https://github.com/walles/moar/blob/master/LICENSE).
Permissive, no attribution required for binaries; only matters
if you redistribute source.

## One Concrete Example

```bash
# 1. read a coloured diff without losing colour
git diff --color=always | moar
#    ↑ less would either strip the colour or show ESC[31m as literal text
#      unless you remembered `less -R`; moar passes ANSI through by default

# 2. search smart-case
#    inside the pager: /error    → matches Error, ERROR, error
#                      /Error    → matches only Error (capital triggers exact)

# 3. drop-in MANPAGER
MANPAGER=moar man kubectl
#    24-bit colour from `man` macros renders cleanly; `h` shows moar's own help

# 4. quit-if-one-screen — behave like cat for tiny inputs
echo "hi" | MOAR='--quit-if-one-screen' moar
#    no full-screen takeover for one-line input; matches `less -F`

# 5. structured-log browsing with colour-coded levels via bat
journalctl -u nginx -n 5000 --no-pager | bat -l log --color=always | moar
#    bat handles syntax highlighting; moar handles paging + smart search

# 6. follow mode (tail -f inside a pager)
moar +F /var/log/syslog
#    Ctrl-C exits follow mode and drops to normal browse, like `less +F`

# 7. open at line N (jump-to from grep output)
rg -n 'panic' src/ | head -1   # → src/main.rs:142:    panic!("…")
moar +142 src/main.rs
```

## Niche It Fills

**The pager you wish `less` was, without retraining.** Every
keybinding most operators actually use in `less` (`/`, `?`, `n`,
`N`, `g`, `G`, `q`, `Space`, `b`, `+F`) works identically in
`moar`; everything that is annoying about `less` defaults
(case-sensitive search, no colour passthrough unless `-R`,
silent failure on `:q`, `--chop-long-lines` confusion) is
fixed in `moar` defaults. Set `PAGER=moar` once and forget.

## Why use it

Five paper-cut fixes that compound across a workday:

1. **ANSI / 24-bit colour passthrough by default.** `git diff`,
   `bat`, `delta`, `rg --color=always`, `kubectl --color`, all
   render correctly with no `--color=auto-detects-pager`
   gymnastics. `less` requires `LESS=-R` (or `-r`) and even
   then 24-bit terminfo can confuse it; `moar` just renders.
2. **Smart-case search.** Lowercase pattern matches case-
   insensitively, any uppercase letter switches to case-
   sensitive — same convention `ripgrep` and modern editors
   use. No `-i` flag to remember.
3. **Single-key quit.** `q` always quits. No `:q` confusion,
   no "Press RETURN" prompt after a search returns no results,
   no "use -X to keep output on screen after quit" archaeology.
4. **Wrap-by-default with continuation marker.** Long lines
   wrap and the right margin shows a `\` so you know it
   wrapped (vs scrolled off). Toggle with `w` if you want
   horizontal scrolling for tabular data.
5. **Built-in `h` help.** Press `h` and the keybindings show
   up in the pager itself — no need to `man less` from another
   shell, no need to remember if it was `?` or `h` or `H` for
   help (different in different `less` builds).

For an LLM-CLI workflow the colour-passthrough alone is the
win: `aichat 'explain this' | bat -l md | moar` keeps the
markdown highlighting all the way to the screen, where `less`
would either strip it or need a startup-time flag dance.

## Vs Already Cataloged

- **Vs [`bat`](../bat/):** orthogonal — `bat` is a `cat`
  replacement that does *syntax highlighting*; it includes a
  built-in pager (uses `less -RF` by default) but is happy to
  pipe to anything via `--paging=never | moar`. The natural
  pairing is `bat file.go | moar` (or
  `BAT_PAGER=moar bat file.go`) — bat colours, moar pages.
- **Vs [`ov`](../ov/):** `ov` is a feature-rich pager + viewer
  that adds column / SQL / follow / multi-document modes — a
  power tool. `moar` is intentionally minimal: a better-defaults
  `less` you set as `$PAGER` and forget. Pick `ov` when you
  want the extra modes (CSV column align, SQL filter); pick
  `moar` for the friction-free everyday pager.
- **Vs [`frogmouth`](../frogmouth/):** different problem —
  `frogmouth` is a Markdown-aware TUI reader (rendered headings,
  links, tables); `moar` is a generic byte/line pager that
  preserves whatever ANSI is already in the stream. Use
  `frogmouth README.md` to *render* markdown; use `moar` to
  *page* the raw stream from `bat` / `git` / `kubectl`.
- **Vs `less(1)` (POSIX classic):** `less` is everywhere by
  default and has 30 years of muscle memory — pick `less` when
  you SSH into a host you do not own. Pick `moar` on machines
  you do own and set `PAGER=moar` for the better defaults.

## Caveats

- **Not 100% keybinding-compatible with `less`.** A handful of
  obscure `less` bindings (mark-set `m<letter>`, multi-file
  navigation `:n` / `:p`, the `&` filter command) are absent
  or differ. The 95% you actually use (`/`, `?`, `n`, `N`,
  `g`, `G`, `q`, `Space`, `b`, `+F`) match.
- **Single-file at a time.** `moar a b c` opens the first
  argument; for multi-file workflows compose with shell
  (`for f in *.log; do moar "$f"; done`) or stay on `less`.
- **No regex flavour switching.** Search is Go's `regexp`
  (RE2) — no PCRE lookarounds, no backreferences. Adequate
  for paging; reach for `rg` upstream if you need richer
  regex inside the file.
- **`$LESS` is *not* read** — `moar` reads `$MOAR` for default
  flags. If your shell rc exports `LESS=-R`, also export `MOAR`
  with whatever subset you want (`--no-statusbar`,
  `--quit-if-one-screen`, `--no-linenumbers`).
- **License is BSD-2-Clause, not MIT.** Functionally identical
  for almost all use; only matters if your distribution policy
  has a MIT-only allowlist (a few do).
- **Truecolour requires a truecolour-capable terminal.** macOS
  `Terminal.app` is 256-colour only; iTerm2 / Ghostty /
  Wezterm / Alacritty / Kitty are fine. `moar` degrades to 256
  colours when the terminal advertises it.
