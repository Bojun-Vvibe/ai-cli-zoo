# caligula

## What it does
A **user-friendly TUI disk imager** in Rust. Replaces the classic
`dd if=image.iso of=/dev/sdX bs=4M status=progress conv=fsync` incantation
with an interactive picker that lists removable block devices, refuses to
clobber the disk you're booted from, validates the source file's hash
(BLAKE3 / MD5 / SHA1 / SHA256 by auto-detecting `*.sha256` / `*.b3`
sidecars or via `--hash`), shows a real-time throughput / ETA progress
bar, and post-write reads the device back to verify byte-for-byte that
the image landed correctly. Handles compressed inputs transparently
(`.xz`, `.gz`, `.bz2`, `.zst`, `.lz4`) by decompressing on the fly so
you don't pre-extract a 4 GB ISO. Single static binary, no daemon, no
GUI toolkit, no Electron — runs over SSH on a headless box.

## Why it's interesting
Different shape from raw `dd` (no progress unless you `kill -USR1`,
silently overwrites whatever you point it at, no verification, no
decompression, footgun-by-default), from balenaEtcher (Electron GUI,
~150 MB, requires a desktop session, can't run over SSH), from
`gnome-multi-writer` / `usbimager` (GUI-only), from Rufus (Windows-only
GUI), and from `pv image.iso | sudo dd of=/dev/sdX` patterns (still
manual device selection, still no verify pass, still no compressed-input
handling). caligula is the *terminal disk imager that does the safety
checks for you* shape: pick it specifically when you flash USB
installers / SD cards regularly on Linux or macOS and want `dd`-class
speed without `dd`-class destruction risk, or when scripting USB
provisioning over SSH on a headless build box. Do **not** pick it for
Windows-only workflows (use Rufus), for partitioning + filesystem-level
operations (use `gparted`, `parted`, `fdisk`), or for cloning live
running systems (use `clonezilla`, `partclone`).

## Niche category
TUI disk imager — interactive `dd` replacement with device picker,
hash verification, transparent decompression, and post-write readback.

## Repo
https://github.com/ifd3f/caligula

## Version pinned
`v0.4.11` (latest tagged release, published 2026-04-11)

## License
- SPDX: `GPL-3.0-or-later`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install caligula

# Cargo (any platform)
cargo install caligula

# Nix (flake provided upstream)
nix profile install github:ifd3f/caligula

# Prebuilt binaries (Linux / macOS, x86_64 + aarch64)
# https://github.com/ifd3f/caligula/releases/tag/v0.4.11
curl -L -o caligula \
  https://github.com/ifd3f/caligula/releases/download/v0.4.11/caligula-x86_64-linux
chmod +x caligula && sudo mv caligula /usr/local/bin/
```

## Usage examples
```sh
# Fully interactive: pick a device from the TUI list, confirm, write, verify
sudo caligula burn ~/Downloads/archlinux-2026.04.01-x86_64.iso

# Headless / scripted: skip the picker, name the device explicitly
sudo caligula burn --root always alpine-3.21-aarch64.iso.xz \
  --output /dev/sdb

# Verify hash before writing (auto-detects archlinux-...iso.sha256 sidecar)
sudo caligula burn ubuntu-24.04.1-desktop-amd64.iso

# Force a specific hash + skip the post-write verify pass (faster, less safe)
sudo caligula burn fedora.iso.zst \
  --hash sha256 \
  --hash-file fedora.iso.zst.sha256 \
  --no-verify

# Decompress a .xz image inline while writing — no temp file on disk
sudo caligula burn raspios-bookworm-arm64-lite.img.xz --output /dev/mmcblk0
```
