# scrcpy

> **Display + control any Android device from the desktop over USB
> or TCP/IP** — a single C client + a `.jar` server pushed onto the
> device via `adb` that streams the framebuffer back as H.264 / H.265
> / AV1 (decoded by the host's FFmpeg, rendered by SDL2) and forwards
> keyboard / mouse / file-drop / clipboard / multi-touch events the
> other way; no root, no Google account, no app install on the
> device, no cloud relay — pinned to **v3.3.4** (released
> 2025-12-17, [LICENSE](https://github.com/Genymobile/scrcpy/blob/v3.3.4/LICENSE),
> SPDX `Apache-2.0`).

Source: <https://github.com/Genymobile/scrcpy>

## TL;DR

Android's official "mirror to PC" story is a fragmented mess:
**Vysor** (commercial, paid tier for HD), **AirDroid** (cloud
relay, account required), **Samsung Flow** / **Phone Link**
(vendor-locked, Windows-only, requires the paired phone to run a
proprietary daemon), `adb screencap` (one-shot PNG, no video),
`scrcpy`'s only real OSS competition is `Genymotion` — and
Genymotion is an *emulator*, not a real-device mirror.

`scrcpy` is the right shape: USB cable, `scrcpy`, window opens.
The whole loop is `adb push scrcpy-server.jar /data/local/tmp/`,
spawn it as `app_process` on the device (no APK install, no
permission grant), open an `adb` reverse-tunnelled socket, stream
H.264 NAL units over it, decode + render with FFmpeg + SDL2 on
the host. Latency is 35–70 ms on a modern Android over USB-3, low
enough to play touch-screen games with mouse + keyboard. The
device-side process dies the moment `scrcpy` exits — there is no
persistent footprint.

## Install

```bash
# macOS
brew install scrcpy

# Debian / Ubuntu
sudo apt install scrcpy

# Arch
sudo pacman -S scrcpy

# Pre-built binaries from upstream tags (Linux x86_64, macOS arm64
# / x86_64, Windows 32 / 64-bit) live at:
#   https://github.com/Genymobile/scrcpy/releases/tag/v3.3.4
```

Hard prereqs: `adb` on `$PATH` (Android platform-tools, Apache-2.0),
`USB debugging` enabled on the device, and the FFmpeg + SDL2
shared libs the binary was built against. macOS Homebrew bottle
pulls those automatically.

## Common invocations

```bash
# Plain mirror over USB
scrcpy

# Wireless (after `adb tcpip 5555 && adb connect 192.168.1.50:5555`)
scrcpy --tcpip=192.168.1.50:5555

# Record the session to mp4 while mirroring
scrcpy --record=demo.mp4

# Cap bitrate + resolution for slow links (8 Mb/s, max 1080p)
scrcpy --video-bit-rate=8M --max-size=1080

# Mirror only — no input forwarding (kiosk / demo mode)
scrcpy --no-control

# Audio forwarding (Android 11+)
scrcpy --audio-codec=opus

# OTG mode — physical keyboard / mouse pass-through with no mirror
scrcpy --otg

# Virtual display (Android 14+) — separate window, the device
# screen stays on the user's own task
scrcpy --new-display=1920x1080
```

## Why orthogonal to existing zoo

The zoo already has dev-loop / file / network / observability /
agent CLIs but **no Android device-mirror or remote-display
client**. `scrcpy` occupies the
"host-controls-a-physical-mobile-device-over-`adb`" niche that
nothing else in this catalog covers — the closest siblings are
`mosh` / `wstunnel` / `sshx` (all *terminal* remoting, not
framebuffer + input), and the framebuffer-streaming TUIs
(`gotty`, `ttyd`) ship the host's terminal *out*, not a remote
device's screen *in*. For mobile QA, on-device repro of a bug
filed by a tester, screen-recording a release demo, or scripting
a phone over `adb` with mouse + keyboard from a laptop, this is
the straight-line answer.

## Caveats

- Apache-2.0 host client; the device-side `scrcpy-server.jar` is
  pushed via `adb` and runs as the `shell` user — it is *not*
  installed as an APK, but on locked-down enterprise devices the
  `adb shell` permission to `app_process` may be disabled (work
  profiles, Knox, MDM-managed devices commonly block this).
- Audio forwarding requires Android 11+; virtual-display
  (`--new-display`) requires Android 14+; older devices fall back
  to mirror-only.
- H.265 / AV1 decode needs an FFmpeg build with the matching
  decoder enabled — Homebrew's `ffmpeg` formula does, but minimal
  Linux distros may not. H.264 always works.
- iOS is **not** supported and never will be — there is no
  equivalent `adb` on iOS. For iOS use `QuickTime` (macOS, USB)
  or paid alternatives like `Reflector`. `scrcpy` is
  Android-only.
- USB-2 cables cap throughput at ~480 Mb/s; for 4K + high-FPS
  capture use a USB-3 cable + a USB-3 host port, or
  `--max-fps=30` to stay headroom-positive.
