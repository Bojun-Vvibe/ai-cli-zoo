# picocom

> **Minimalist serial-port terminal emulator** — opens a tty
> device (`/dev/ttyUSB0`, `/dev/cu.usbserial-*`,
> `/dev/ttyACM0`, etc.), passes keystrokes through to the
> remote serial peer (microcontroller, router console, U-Boot
> prompt, RS-485 sensor), and prints the peer's bytes back to
> your terminal — with a ~150 KB binary, zero dependencies
> beyond libc, and one keybind family (`Ctrl-A` followed by a
> letter) for runtime ops (change baud, toggle local echo,
> send file via `sx`/`sz`/`sb`/`ax`, exit cleanly). Pinned to
> **v3.1** (released 2017-12-05,
> [LICENSE.txt](https://github.com/npat-efault/picocom/blob/3.1/LICENSE.txt),
> GPL-2.0).

Source: <https://github.com/npat-efault/picocom>

## TL;DR

Hardware bring-up, embedded debugging, router console rescue,
and IoT sensor wiring all converge on the same workflow: a USB-
serial cable into your laptop, a baud rate, and a tty that
needs a terminal emulator. The space has three tiers: (a) `cu`
(BSD), `screen /dev/ttyUSB0 115200`, `tio`, `microcom` — basic
terminals, varying ergonomics; (b) `minicom` — feature-rich
ncurses TUI with menus, scripts, and a captive UX that catches
newcomers (the famed "how do I exit minicom" StackOverflow
thread); (c) `picocom` — sits between (a) and (b): no curses,
no captive UX, no dependencies, but enough features to be the
daily driver — runtime baud changes, hardware/software flow
control, file send/receive via external `sx`/`sz`/`sb`/`ax`
helpers, hex-mode, and a clean `Ctrl-A Ctrl-X` exit. The
binary is small enough to drop onto a recovery USB stick or a
constrained embedded host.

## Install

```bash
# macOS (Homebrew)
brew install picocom

# Debian / Ubuntu
sudo apt-get install -y picocom

# Arch
sudo pacman -S picocom

# Fedora
sudo dnf install picocom

# From source — trivial, no autotools
git clone --branch 3.1 https://github.com/npat-efault/picocom.git
cd picocom && make && sudo install -m 0755 picocom /usr/local/bin/

# Verify
picocom --help | head -1    # picocom v3.1
```

On Linux, your user must be in the `dialout` (Debian/Ubuntu)
or `uucp` (Arch) group to read/write `/dev/tty*` without
`sudo`:

```bash
sudo usermod -aG dialout $USER && newgrp dialout
```

## License

GPL-2.0 — see
[LICENSE.txt](https://github.com/npat-efault/picocom/blob/3.1/LICENSE.txt).
Copyleft (v2): redistributing modified `picocom` requires
publishing source under GPL-2.0. Personal use and
distribution of unmodified binaries are unrestricted.

## Common invocations

```bash
# Open a serial port at 115200-8N1, the embedded default
picocom -b 115200 /dev/ttyUSB0
# macOS uses /dev/cu.* nodes
picocom -b 115200 /dev/cu.usbserial-0001
# Arduino / micro:bit / ESP32 typically expose ACM nodes
picocom -b 115200 /dev/ttyACM0

# Custom framing — 7E1 with hardware flow, common on legacy lab gear
picocom -b 9600 -d 7 -p 1 -y e -f h /dev/ttyUSB0

# Echo locally what you type (off by default — saves the dual-print problem
# when the remote echoes back)
picocom -b 115200 --echo /dev/ttyUSB0

# IMAP terminal? No. But this is the standard recipe for U-Boot rescue
picocom -b 115200 --imap=lfcrlf,crcrlf --omap=crlf /dev/ttyUSB0

# Hex-mode — bytes shown as 0xNN, useful for protocol bring-up
picocom -b 9600 --imap=spchex /dev/ttyUSB0

# Pipe a file in (one-shot send), then drop into interactive mode
echo "boot" | picocom -b 115200 --noinit --noreset /dev/ttyUSB0

# Inside a session, the only keybind family you need:
#   Ctrl-A Ctrl-X    exit picocom
#   Ctrl-A Ctrl-S    send file (xmodem/ymodem/zmodem via sx/sb/sz)
#   Ctrl-A Ctrl-R    receive file
#   Ctrl-A Ctrl-U    raise baud
#   Ctrl-A Ctrl-D    lower baud
#   Ctrl-A Ctrl-F    toggle hardware flow control
#   Ctrl-A Ctrl-Y    toggle software flow control
#   Ctrl-A Ctrl-V    show current settings
#   Ctrl-A Ctrl-C    toggle local echo
#   Ctrl-A Ctrl-T    toggle hex-mode
```

## Why use it

- **Tiny, dependency-free, copy-paste deployable.** ~150 KB
  static binary; one C source tree, plain `make`, no
  autotools / CMake / ncurses. Drops onto a recovery image
  or a router with `scp` and runs.
- **Not captive.** Unlike `minicom`, there is no full-screen
  TUI to escape from. `picocom` is a transparent passthrough;
  `Ctrl-A Ctrl-X` exits, the terminal returns to your shell
  unmodified. New users do not get stranded.
- **Runtime baud / flow / mode changes.** `Ctrl-A Ctrl-U` /
  `Ctrl-A Ctrl-D` walk the baud-rate ladder live;
  `Ctrl-A Ctrl-F` toggles RTS/CTS without restart. Critical
  on bring-up when the bootloader speaks 115200 and the
  application speaks 921600.
- **Honest file-transfer story.** `picocom` does not
  reimplement xmodem/ymodem/zmodem; it shells out to
  `sx`/`sb`/`sz`/`ax` (lrzsz). Configurable via `--send-cmd`
  / `--receive-cmd` so any external tool works.

## Vs Already Cataloged

- (None of the directly adjacent serial-terminal tools —
  `tio`, `minicom`, `screen`, `cu`, `microcom` — are in the
  zoo as of this entry; this is a fresh niche.)
- **Vs [`mosh`](../mosh/) / [`tmate`](../tmate/) /
  [`upterm`](../upterm/):** wrong layer. Those are remote
  *shell* / *session-sharing* tools over TCP; `picocom`
  speaks bytes-on-a-wire over a tty. Adjacent niche, no
  overlap.
- **Vs [`scrcpy`](../scrcpy/):** `scrcpy` mirrors an Android
  device over USB-or-WiFi (display + input). `picocom`
  drives a serial console on USB-tty. Both touch USB,
  nothing else in common.
- **Vs [`websocat`](../websocat/):** `websocat` is the
  WebSocket-as-stdio bridge; `picocom` is the
  serial-port-as-stdio bridge. Same shape (transparent
  byte-level pipe), different transport.
- **Vs [`tmux`](https://github.com/tmux/tmux) /
  [`zellij`](../zellij/) /
  [`dvtm`](../dvtm/):** complementary. Run `picocom` *inside*
  a `tmux` / `zellij` / `dvtm` pane so the serial console
  survives an SSH disconnect; the multiplexer handles
  detach/reattach, `picocom` handles the tty.

## Caveats

- **Permissions.** On Linux you need group access to
  `/dev/tty*` (`dialout` / `uucp`); without it, you'll
  get `cannot open /dev/ttyUSB0: Permission denied` and
  reach for `sudo`, which is the wrong long-term answer.
- **Release cadence is glacial.** v3.1 dates from
  2017-12-05. The serial-terminal problem is solved and
  the project is effectively in maintenance; bugs get
  fixed on `master` but tagged releases are infrequent.
  For a more actively maintained alternative with a
  similar minimal philosophy, look at `tio` (separate
  project, MIT-licensed, semver releases).
- **No scripting language.** `minicom` has a runscript
  language (`runscript`); `picocom` has none. For
  scripted serial dialogues use `expect` /
  `pexpect` driving `picocom`, or jump to a richer tool.
- **No session logging by default.** Use `--logfile
  session.log` to capture; otherwise scrollback lives
  only in your terminal emulator's buffer.
- **macOS device names differ.** Use `/dev/cu.usbserial-*`
  (call-out, the right choice for outbound serial), not
  `/dev/tty.usbserial-*` (dial-in, will block waiting
  for carrier detect).
- **Hardware flow control assumes the cable carries
  RTS/CTS.** Many cheap USB-serial dongles only wire
  TX/RX/GND; `-f h` then silently does nothing. Verify
  with `Ctrl-A Ctrl-V` (show settings) and your cable's
  pinout.
