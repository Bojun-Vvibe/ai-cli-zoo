# qrcp

> **Transfer files between your computer and your phone over the
> local Wi-Fi by scanning a QR code in the terminal** — `qrcp`
> spins up a one-shot HTTP server bound to your LAN interface,
> generates a URL that points at either a "download this file" or
> "upload to this directory" handler, renders that URL as an
> ASCII / unicode QR code in your terminal, waits for the phone
> to hit the URL exactly once, then shuts the server down. No
> account, no cloud round-trip, no AirDrop / Nearby Share / Quick
> Share platform lock-in. Pinned to **v0.11.6**
> ([LICENSE](https://github.com/claudiodangelis/qrcp/blob/main/LICENSE),
> MIT).

Source: <https://github.com/claudiodangelis/qrcp>

## TL;DR

`qrcp send ./file.pdf` prints a QR; phone camera opens the URL;
file lands in `~/Downloads`; server exits. `qrcp receive` flips
the direction — pick a destination dir on the laptop, scan the
QR on the phone, get a tiny upload form, push files back. Single
static Go binary, zero config for the happy path, autodetects the
right network interface.

## Install

```bash
# Homebrew (macOS / Linux)
brew install qrcp

# Go
go install github.com/claudiodangelis/qrcp@v0.11.6

# Pre-built release tarball
curl -LO https://github.com/claudiodangelis/qrcp/releases/download/v0.11.6/qrcp_0.11.6_macos_arm64.tar.gz
tar xf qrcp_0.11.6_macos_arm64.tar.gz
sudo install qrcp /usr/local/bin/

# Linux package managers
# Arch (AUR): yay -S qrcp-bin
# Nix:        nix-env -iA nixpkgs.qrcp

# verify
qrcp version    # 0.11.6
```

First run prompts you to pick a network interface and writes
`~/.qrcp.json` (overrideable with `--config`); subsequent runs
reuse it silently.

## License

MIT — see
[LICENSE](https://github.com/claudiodangelis/qrcp/blob/main/LICENSE).
Permissive, redistribute and modify freely.

## One Concrete Example

```bash
# 1. send a file from laptop -> phone
qrcp send ~/Downloads/report.pdf
# prints a QR for http://192.168.1.42:53827/send/abc123 — scan,
# tap "download", server exits when the byte count completes.

# 2. send multiple files (auto-zipped on the fly)
qrcp send report.pdf invoice.pdf cover.png

# 3. receive from phone -> laptop, into a chosen directory
qrcp receive --output ~/Inbox

# 4. force a specific interface (multi-NIC laptop, VPN up)
qrcp send --interface en0 ./big.iso

# 5. force HTTPS with a self-signed cert (older Android cameras)
qrcp send --secure --tls-cert cert.pem --tls-key key.pem ./file

# 6. keep the server up for N transfers instead of one-shot
qrcp send --keep-alive ./shared.tar.gz   # Ctrl-C to stop
```

## Niche It Fills

**Cross-vendor, cross-OS, no-cloud "AirDrop" between any laptop
and any phone with a camera.** AirDrop is Apple-only, Quick Share
needs a Google account on both sides and a recent Android,
Bluetooth file transfer is slow and flaky, and uploading to
Dropbox / Drive / WeTransfer means a round-trip through someone
else's server for a file you already have locally. `qrcp` answers
the same question — "get this file from device A to device B
without cables" — using only the LAN you are already on, with no
account on either side and no daemon left running.

## Why use it

1. **The QR is the auth.** The randomly-generated path segment
   in the URL (`/send/<random>`) is the only way to hit the
   handler; the server binds to the LAN interface only (not
   public), serves exactly one request by default, then exits.
   You don't have to think about ACLs.
2. **Bidirectional.** `send` is laptop-to-phone, `receive` is
   phone-to-laptop with an HTML upload form rendered server-side
   — same UX, same QR-as-pairing pattern.
3. **One static binary, no daemon.** Nothing runs in the
   background between transfers; no menu-bar app, no system
   service, no cloud account. Goes away cleanly on shared / work
   machines.

For a CLI / scripting workflow `qrcp send "$artifact"` is a
useful "out" from a build pipeline when the next consumer is a
human with a phone (e.g. mobile QA needs the latest APK on
device, designer needs the latest export on iPad) and the file
is too big for chat / email.

## Vs Already Cataloged

- **Vs [`croc`](../croc/) / [`magic-wormhole`](../magic-wormhole/):**
  both are the "PAKE-secured peer-to-peer file transfer over the
  internet" pattern with a short shared code phrase as auth and
  a relay server in the middle for NAT traversal. `qrcp` is the
  LAN-only, QR-as-code variant: no relay, no internet, no code
  to type — but both endpoints must be on the same Wi-Fi /
  network. Use `croc` / `magic-wormhole` when the receiver is
  remote; use `qrcp` when the receiver is the phone in your
  pocket.
- **Vs [`sharik`](../sharik/) / [`piknik`](../piknik/):** sharik
  is a GUI-first Android app for the same LAN-share niche;
  `piknik` is the multi-device clipboard / paste-bin pattern
  (text + small files via a relay you run). `qrcp` is the
  terminal-first, no-relay, QR-pairing point in that design
  space.
- **Vs `python3 -m http.server` / `caddy file-server`:** the
  ad-hoc HTTP-server pattern works but has none of the
  ergonomics — you have to figure out your LAN IP, type the URL
  on the phone, expose the *whole* directory, and remember to
  Ctrl-C. `qrcp` does the IP discovery, the random path, the
  one-shot lifecycle, and the QR rendering, all from one
  command.

## Caveats

- **Same network only.** Both devices must be on the same LAN
  (or routable subnet). Corporate Wi-Fi with client isolation
  (guest networks, hotel Wi-Fi) blocks the connection — switch
  to a phone hotspot if both devices can join it.
- **HTTP by default.** Plain HTTP is fine for a 30-second
  one-shot transfer on your home LAN; on hostile networks
  consider `--secure` with a self-signed cert (you will have to
  accept the warning on the phone). Some recent Android camera
  apps refuse `http://` URLs and demand `https://`.
- **No resume, no integrity check.** Transfers are a single HTTP
  request; if the phone walks out of Wi-Fi range mid-transfer
  the file is truncated and the server still exits clean. For a
  multi-GB transfer over flaky Wi-Fi, fall back to `croc` (which
  resumes).
- **Interface autodetect picks the wrong NIC sometimes.** Common
  on laptops with VPN up (`utun*` on macOS, `tun0` on Linux).
  Pin with `--interface` or set `interface` in `~/.qrcp.json`.
- **Receive form is unauthenticated for the lifetime of the
  one-shot.** Anyone on the LAN who scans the QR (or guesses the
  random path) can upload until the configured file count is
  reached; do not leave `--keep-alive` running unattended on a
  shared network.
