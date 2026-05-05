# kanata

> **Cross-platform keyboard remapper** that turns any keyboard
> into a programmable layout with tap/hold modifiers,
> homerow mods, layers, leader sequences, macros, and Caps-Lock
> as Escape — driven by a single Lisp-shaped config file,
> running as a userspace daemon on Linux, macOS, and Windows.
> Pinned to **v1.11.0** (SPDX: `LGPL-3.0`,
> [LICENSE](https://github.com/jtroo/kanata/blob/main/LICENSE)).

Source: <https://github.com/jtroo/kanata>

## TL;DR

`kanata` reads keyboard events from the OS, runs them through a
declarative remap engine, and emits remapped events back to the
OS via a virtual keyboard device — so an ortholinear-friendly
home-row-mods + layers + tap-hold layout that previously
required QMK firmware on a custom keyboard now works on the
laptop's built-in keyboard. One config file, one Rust daemon,
identical layout across Linux + macOS + Windows.

## Install

```bash
# Cargo (works on all three platforms)
cargo install kanata --locked

# Homebrew (macOS)
brew install kanata

# Pre-built binary
curl -LO "https://github.com/jtroo/kanata/releases/download/v1.11.0/kanata"
chmod +x kanata
sudo install kanata /usr/local/bin/

# verify
kanata --version    # kanata 1.11.0
```

OS-specific setup:
- **Linux:** kanata reads from `/dev/input/event*` (needs
  `CAP_SYS_ADMIN` or `uinput` group membership) and writes via
  `uinput`.
- **macOS:** depends on the Karabiner-DriverKit virtual HID
  device (the upstream project's documented prerequisite).
- **Windows:** uses the Interception driver.

## License

LGPL-3.0 — see
[LICENSE](https://github.com/jtroo/kanata/blob/main/LICENSE).

## One Concrete Example

```lisp
;; ~/.config/kanata/config.kbd
;; Caps-Lock as Escape on tap, Ctrl on hold;
;; A/S/D/F as home-row Shift/Ctrl/Alt/Cmd on hold;
;; layer 2 under spacebar for navigation arrows on hjkl.

(defcfg
  process-unmapped-keys yes)

(defsrc
  caps a    s    d    f    spc  h    j    k    l)

(deflayer base
  @cesc @ash @asc @ada @afm @nav h    j    k    l)

(deflayer nav
  _    _    _    _    _    _    left down up   rght)

(defalias
  cesc (tap-hold 200 200 esc lctl)        ;; Caps  : tap=Esc, hold=Ctrl
  ash  (tap-hold 200 200 a   lsft)        ;; A     : tap=a,   hold=Shift
  asc  (tap-hold 200 200 s   lctl)        ;; S     : tap=s,   hold=Ctrl
  ada  (tap-hold 200 200 d   lalt)        ;; D     : tap=d,   hold=Alt
  afm  (tap-hold 200 200 f   lmet)        ;; F     : tap=f,   hold=Cmd/Win
  nav  (layer-while-held nav))             ;; Space : hold = nav layer
```

```bash
# 1. start kanata with the config above
sudo kanata --cfg ~/.config/kanata/config.kbd

# 2. live-reload on file change (no daemon restart)
sudo kanata --cfg ~/.config/kanata/config.kbd --watch

# 3. validate the config without starting the keyboard hook
kanata --cfg ~/.config/kanata/config.kbd --check

# 4. multiple layers + dynamic macros via TCP control port
sudo kanata --cfg ~/.config/kanata/config.kbd --port 12345
# elsewhere:
echo '{"ChangeLayer":{"new":"nav"}}' | nc 127.0.0.1 12345

# 5. systemd unit on Linux for automatic boot start
sudo cp packaging/kanata.service /etc/systemd/system/
sudo systemctl enable --now kanata
```

## Niche It Fills

**Programmable-keyboard ergonomics on stock hardware.** Without
kanata: either buy a custom keyboard and flash QMK / ZMK
firmware (hardware lock-in, doesn't help on the laptop's
built-in keys), or use platform-specific tools — Karabiner on
macOS, `xremap` / `interception-tools` on Linux, AutoHotKey
on Windows — each with a different config dialect, none
covering tap-hold + layers + leader sequences with the
fidelity QMK does. With kanata: one Lisp-shaped config,
identical behavior on all three OSes.

## Why use it

1. **Layers and tap-hold are first-class.** Home-row mods,
   space-as-layer, leader sequences, dynamic macros — the
   ergonomic-keyboard primitives that previously required
   custom firmware now work on any keyboard.
2. **One config, three OSes.** The same `.kbd` file works on
   Linux, macOS, and Windows — config goes in dotfiles, not
   per-OS settings panels.
3. **Live-reload.** `--watch` reloads on file save — iterate
   on a layer design in real time, no daemon restart.
4. **TCP control port.** External tools can switch layers,
   trigger macros, or query state — wire to a window manager
   (`kanata layer set work` when Slack focuses, `kanata layer
   set vim` when the editor focuses).
5. **Validates statically.** `--check` parses the config and
   reports unresolved aliases / overlapping bindings without
   touching the keyboard — safe to wire into a CI gate over
   your dotfiles repo.
6. **Open Lisp-shaped DSL.** s-expressions you can write by
   hand and diff in a PR — not a binary blob, not a GUI-only
   format.

## Vs Already Cataloged

- **Vs [`espanso`](../espanso/):** `espanso` is text expansion
  ("`:eml` -> my email address"); kanata is keyboard remapping
  ("Caps Lock acts as Escape when tapped, Ctrl when held").
  Different layers of the input stack — they compose: kanata
  delivers an unambiguous keystream that espanso then expands.
- **Vs Karabiner-Elements (macOS-only):** Karabiner is the
  reference for macOS-only remapping with a GUI config and
  the same DriverKit dependency kanata uses on macOS. Pick
  Karabiner if you only ever work on macOS and prefer a GUI;
  pick kanata if your config needs to work on a Linux laptop
  too.
- **Vs `xremap` / `interception-tools` (Linux-only):** Same
  category, Linux only. kanata covers the same layout space
  with cross-platform reach.
- **Vs QMK / ZMK firmware:** Those run on the keyboard's MCU
  — best for custom keyboards you own. kanata is the
  software-only answer for built-in laptop keyboards and
  off-the-shelf USB keyboards.
- **Vs OS accessibility settings (Sticky Keys etc):** Those
  cover the simple cases (one key remap, sticky modifiers).
  kanata covers the rest of the layout-language surface.

## Caveats

- **Requires elevated privileges.** Reading raw input devices
  needs `CAP_SYS_ADMIN` (Linux), DriverKit consent (macOS),
  or the Interception driver install (Windows). Plan the
  install on a managed device accordingly.
- **macOS DriverKit dep is non-trivial.** Karabiner-DriverKit
  installation requires a reboot and a System Settings allow
  step. One-time, but document it for new contributors.
- **Layer state is in-process.** Restarting the daemon resets
  to the base layer — long leader sequences mid-restart are
  lost. Use the TCP control port to script layer state if you
  need persistence across restarts.
- **LGPL-3.0.** Linking against kanata internals from a
  proprietary tool requires LGPL compliance. Using the binary
  via its config file or TCP port has no such requirement.
- **Tap-hold tuning is a craft.** A 200 ms timeout that feels
  natural for one typist will feel wrong for another. Plan to
  iterate on `--watch` for a few days before settling.
