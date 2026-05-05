# cyme

> **A modern `lsusb` rewritten in Rust with structured
> output, tree rendering, and a Mac/Linux/Windows backend
> matrix** — a single ~3 MB binary that enumerates every
> USB device on the host, prints a colourised tree (or
> JSON) with vendor / product / class / speed / power
> data, watches for hotplug events, and can decode
> per-class descriptors (HID report descriptors, audio
> control units, video format blocks) the way `lsusb -v`
> can on Linux but cannot on macOS. Pinned to **v2.3.0**
> (commit `65806e4578b863eb3a963bfb893a699e846ea4df`,
> [LICENSE](https://github.com/tuna-f1sh/cyme/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/tuna-f1sh/cyme>

## TL;DR

`cyme` is the answer to "I have a USB-C dock, three
peripherals, and a hub-of-hubs and I need to know
*exactly* what is plugged where, what speed each link
negotiated, what device class each endpoint claims, and
how much bus power each draws — on macOS, where
`system_profiler SPUSBDataType` is the only built-in
option and its plist output is not pipeline-friendly."
The default invocation (`cyme`) prints a tree that looks
like `lsusb -t` but with the device names from the USB
ID database resolved in-line, the negotiated speed and
power coloured by class, and per-device VID:PID columns
ready to grep. Output formats include the default
human-readable tree, `--json` (every field of the libusb
descriptor, suitable for `jq` pipelines), `--lsusb`
(byte-compatible with `lsusb -v` for scripts that already
parse that), and `--watch` (a TUI mode that updates on
hotplug events). The backend is `nusb` (pure-Rust libusb
replacement) on Linux and Windows and the IOKit bridge
on macOS, so the tool works on a stock macOS machine
without a Homebrew `libusb` install.

## Install

```bash
# Homebrew (macOS / Linux)
brew install cyme

# Cargo
cargo install --locked cyme

# Linux package managers
# Arch (AUR): yay -S cyme
# Nix: nix-env -iA nixpkgs.cyme
# Debian / Ubuntu: download .deb from the GitHub Releases page

# from a release tarball (any OS)
curl -Lo cyme.tar.gz "https://github.com/tuna-f1sh/cyme/releases/download/v2.3.0/cyme-aarch64-apple-darwin.tar.gz"
tar xf cyme.tar.gz
sudo install cyme /usr/local/bin/

# verify
cyme --version    # cyme 2.3.0
```

No daemon, no kernel module, no setuid bit — on Linux the
binary needs read access to `/dev/bus/usb/*` (typically
granted to users in the `plugdev` group) for full
descriptor enumeration; without it, `cyme` falls back to
the basic device list visible to unprivileged users.

## License

GPL-3.0 — see
[LICENSE](https://github.com/tuna-f1sh/cyme/blob/main/LICENSE).
Linking and redistribution allowed under copyleft terms;
the binary itself is freely runnable on any host.

## One Concrete Example

```bash
# 1. show the USB tree (default invocation)
cyme

# 2. JSON pipeline — every device with VID 0x05ac (Apple)
cyme --json | jq '.devices | .. | objects | select(.vendor_id == 1452)'

# 3. only show devices on USB 3.x (5 Gbps or faster)
cyme --filter-class hub --hide-buses --headings | grep -E '5\.0 Gb|10\.0 Gb|20\.0 Gb'

# 4. watch hotplug events (TUI; Ctrl-C to exit)
cyme --watch

# 5. drop into the lsusb-compatible verbose mode for one device
cyme --lsusb --verbose --device 05ac:8600

# 6. dump everything one device exposes, including HID report descriptors
cyme --device 046d:c539 --more --decode-hid

# 7. JSON snapshot for diffing across reboots
cyme --json > /tmp/usb-before.json
# ... reboot, replug ...
cyme --json > /tmp/usb-after.json
diff <(jq -S . /tmp/usb-before.json) <(jq -S . /tmp/usb-after.json)
```

## Niche It Fills

**`lsusb` for the cross-platform / structured-output
era.** The classic `lsusb` is Linux-only and emits text
designed for human eyeballs; `system_profiler
SPUSBDataType` is macOS-only and emits a verbose plist
that nobody wants to parse. `cyme` covers all three
desktop OSes from one Rust binary, defaults to a
colourised tree that's readable at a glance, and emits
JSON when a script needs to consume the data. For an
agent or automation that needs to answer "is the
microcontroller plugged in and on USB-FS or USB-HS?" the
right shape is `cyme --json | jq '...'` rather than
shelling out to a different command per OS.

## Why use it

Three things `cyme` does that the built-ins do not:

1. **Structured output as a first-class verb.**
   `--json` is a stable contract that exposes every
   libusb descriptor field — VID, PID, class, subclass,
   protocol, speed, power, port path, every interface,
   every endpoint. Scripts and agents can `jq` against
   it instead of regex-parsing `lsusb -v`'s indented
   text format or XML-parsing `system_profiler`'s plist.
2. **Cross-platform parity from one binary.** The same
   `cyme` invocation works on a Linux CI runner, a
   developer's macOS laptop, and a Windows test bench;
   the backend (`nusb` on Linux/Windows, IOKit on macOS)
   is invisible to the caller. No per-OS branching in
   shell scripts.
3. **Tree + filter + watch in one tool.** `lsusb -t`
   shows the topology but not vendor names; `lsusb -v`
   shows the descriptors but not the topology; neither
   watches for hotplug. `cyme` covers all three modes
   with shared filtering flags (`--filter-class`,
   `--filter-vid`, `--filter-pid`, `--device`), so a
   single mental model gets you from "show me the tree"
   to "show me only Logitech devices on USB-HS or
   faster, and update when I plug something in".

For an LLM-CLI workflow that orchestrates hardware-test
fixtures or USB peripherals, `cyme --json | jq
'.devices[].name'` is a deterministic plain-text answer
to "what is connected right now" that the agent can
reason about without parsing a different format per host
OS.

## Vs Already Cataloged

- **Vs [`fastfetch`](../fastfetch/) /
  [`hyfetch`](../hyfetch/) /
  [`macchina`](../macchina/):** orthogonal — those are
  one-shot system-info banners (CPU / GPU / OS / RAM /
  uptime); `cyme` is a deep, structured USB-bus
  inspector. The fetch tools mention USB only in
  passing (often just "N USB devices detected"); `cyme`
  is the next call when you actually need to *see* what
  those N devices are.
- **Vs [`bottom`](../bottom/) / [`btop`](../btop/) /
  [`glances`](../glances/):** orthogonal — those are
  live system monitors (CPU, memory, disk, network);
  USB topology is static enough that it doesn't belong
  in a polling dashboard. Use a monitor for "is this
  process eating CPU"; use `cyme` for "is this device
  enumerated and at what speed".
- **Vs `lsusb` (usbutils, not cataloged):** `cyme` is
  a strict superset on the output side — same `-t` tree
  shape, same `-v` verbose option (via `--lsusb -v`),
  plus JSON, watch mode, cross-platform parity, and
  per-class decoders that mainline `lsusb` only
  partially implements. On Linux, `lsusb` is still the
  smaller dependency for one-shot scripts; `cyme` wins
  the moment you need JSON or macOS support.
- **Vs `system_profiler SPUSBDataType` (macOS,
  built-in, not cataloged):** the native macOS option
  emits an XML plist that requires `plutil -convert
  json -` to feed into `jq` and lacks tree formatting.
  `cyme` emits the same data in a flat, documented JSON
  schema with a human-readable tree as the default —
  one binary, two output modes, no `plutil` round-trip.

## Caveats

- **GPL-3.0, not MIT/Apache.** Embedding `cyme` source
  in a proprietary binary requires care; using it as a
  CLI tool in any pipeline (open or closed) is fine.
  The license affects redistribution of the source
  tree, not script invocation.
- **macOS backend gives shallower descriptors than
  Linux.** IOKit exposes the device tree but not every
  byte of every interface descriptor that
  `/dev/bus/usb/*` does on Linux. `--lsusb -v`-style
  full dumps are richer on Linux than on macOS for the
  same hardware. Class-specific decoders (HID, audio,
  video) work on both but reach different depths.
- **Some USB hubs lie about negotiated speed.** A
  device plugged into a multi-tier hub may report the
  slowest link in its chain rather than its own
  capability; this is a USB-spec / hub-firmware artefact,
  not a `cyme` bug. Cross-check with `--more` to see
  the per-port path and `--device VID:PID --verbose` to
  see what the device itself claims.
- **No Bluetooth, no PCI, no Thunderbolt deep
  inspection.** Despite USB-C / TB4 sharing connectors,
  `cyme` is USB-only — Thunderbolt-attached PCIe
  devices are out of scope. For PCI use `lspci`; for
  Thunderbolt use `boltctl` (Linux) or `system_profiler
  SPThunderboltDataType` (macOS).
- **Hotplug watch on macOS depends on IOKit
  notifications.** Some USB-C dock chipsets (notably
  some DisplayLink / VIA combos) suppress detach events
  to the OS; `cyme --watch` will miss those until the
  next attach. Workaround: poll `cyme --json` on a
  timer instead.
