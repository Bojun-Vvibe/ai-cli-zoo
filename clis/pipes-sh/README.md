# pipes-sh

> **Animated pipes terminal screensaver in pure Bash.**
> The classic Windows 95 3D-pipes screensaver, reimplemented as a
> ~300-line shell script that draws colored pipe segments using
> nothing but ANSI escapes and box-drawing characters.
> Pinned to **v1.3.0**
> ([LICENSE.md](https://github.com/pipeseroni/pipes.sh/blob/master/LICENSE.md),
> MIT).

Source: <https://github.com/pipeseroni/pipes.sh>

## TL;DR

`pipes.sh` fills your terminal with growing, turning, randomly
colored pipes until you press `q`. It exists because (a) it's
charming, (b) it's a useful "is this terminal still alive" canary
for long-running SSH sessions, and (c) it's a remarkable
demonstration of how much you can do in plain Bash with
`tput`/ANSI alone — no curses, no compiled binary.

## Install

```bash
# Homebrew
brew install pipes-sh

# Arch
pacman -S pipes.sh

# Debian / Ubuntu
apt install pipes.sh

# From source (single script)
git clone https://github.com/pipeseroni/pipes.sh ~/src/pipes.sh
sudo make -C ~/src/pipes.sh install PREFIX=/usr/local

# verify
pipes.sh -v   # pipes.sh 1.3.0
```

## License

MIT — see
[LICENSE.md](https://github.com/pipeseroni/pipes.sh/blob/master/LICENSE.md).
Permissive; embed in dotfiles, login shells, motd, anywhere.

## One Concrete Example

```bash
# Plain run: one pipe, default colors, until you press q
pipes.sh

# Four pipes, ASCII-only character set (no Unicode box drawing),
# bold on, reset every 2000 frames so the screen doesn't fill up
pipes.sh -p 4 -t 2 -B -r 2000

# Use as a screen-lock placeholder while you grab coffee:
#   tmux split-window -h 'pipes.sh -p 6 -f 75'
# (-f 75 = 75 frames/sec; default is ~75 too)

# As a terminal-alive canary on a noisy SSH session
ssh prod-jumphost 'pipes.sh -p 2'
# If the connection dies, the pipes stop moving — instantly visible.
```

## Niche It Fills

**The "I want a zero-dependency, instantly-recognizable terminal
animation that runs over SSH and proves the link is alive" gap.**
Most terminal toys need a runtime (Python, Node, Go) or a curses
library. `pipes.sh` is one Bash script with no dependencies beyond
`tput`. It runs on every Linux box, every BSD, every macOS install,
inside every container image, over every SSH hop.

## Why use it

1. **Pure Bash, zero deps.** Works on a fresh Alpine container, on
   a router's BusyBox shell, on a recovery initramfs — anywhere
   `bash` and `tput` exist.
2. **Tiny attack surface.** It's ~300 lines you can read in five
   minutes. No package-tree to audit.
3. **Genuinely useful as a liveness canary.** Pipes that stop
   moving = your SSH / serial / mosh session is wedged. Easier to
   eyeball than `top`.

## Vs Already Cataloged

- **Vs [`cmatrix`](../cmatrix/):** `cmatrix` is the green-rain
  Matrix screensaver in C with ncurses. Same vibe (terminal
  screensaver), different aesthetic, different dependency
  footprint — `pipes.sh` needs no compilation.
- **Vs [`hollywood`](../hollywood/) (if cataloged):** `hollywood`
  is a multi-pane "fake hacker" composition. `pipes.sh` is the
  single-effect, single-pane minimalist counterpart.

## Caveats

- **Bash 4+.** Uses associative arrays and modern parameter
  expansion. Default macOS `/bin/bash` is 3.2 — install GNU bash
  via Homebrew (or just use the Homebrew-installed `pipes.sh`
  wrapper, which picks the right interpreter).
- **Unicode box drawing needs a UTF-8 locale.** With `-t 2` (ASCII
  set) it works on legacy or broken-locale terminals; the default
  set assumes `LANG=*.UTF-8`.
- **Burns a CPU core if you crank `-f`.** It's a busy-loop
  animation. Defaults are fine; `-f 200` will spin a core.
- **Not a real screensaver.** It does not lock the terminal or
  blank on input. `q` quits, any other key passes through. Pair
  with `vlock` / `physlock` if you actually need to lock.
