# syncthing

> **A continuous, peer-to-peer file synchronization daemon — your
> files stay on devices you control, no cloud account, no
> third-party server holding the bytes.** TLS-mutual-auth between
> devices, automatic discovery on the LAN, NAT-traversal via a
> public relay pool, conflict copies instead of silent overwrites,
> and a local web UI on `127.0.0.1:8384` for management. Pinned to
> **v2.0.16** (commit
> `14e8020aa52e3f493f651cca20413ab67a626e2c`,
> [LICENSE](https://github.com/syncthing/syncthing/blob/main/LICENSE),
> MPL-2.0).

Source: <https://github.com/syncthing/syncthing>

## TL;DR

`syncthing` is what you reach for when Dropbox / iCloud / OneDrive
are off the table — because the data is sensitive, because the
files are too large for a free tier, because you do not want a
vendor seeing the names of your files, or because you want a
sync relationship that survives a vendor's bankruptcy. Run
`syncthing` on each device (laptop, desktop, NAS, phone via the
F-Droid client). Each daemon prints its **device ID** (a long
base-32 string derived from its self-signed cert). Paste device A's
ID into device B's web UI to add the trust relationship, then
share a folder by path on each side — the daemons negotiate over
TLS, discover each other on the LAN via mDNS or over the internet
via the public discovery / relay pool, and start syncing changes
within seconds. The 2.0 line (April 2026) replaces the LevelDB
backend with SQLite (faster, less buggy, easier to inspect),
switches logging to structured key-value entries, and ships
multi-connection (3 by default) for higher throughput on long-fat
links. The cost is operational: you run a daemon per device, you
do conflict resolution yourself when two sides edit the same file
offline (syncthing creates `file.sync-conflict-<date>-<id>.ext`
copies), and the web UI is the management surface — there is no
mobile-quality desktop app.

## Install

```bash
# Homebrew (macOS / Linux)
brew install syncthing

# Linux package managers
# Debian/Ubuntu: apt install syncthing  (or use the apt.syncthing.net repo for current)
# Arch:          pacman -S syncthing
# Fedora:        dnf install syncthing
# Nix:           nix-env -iA nixpkgs.syncthing

# Docker
docker run -d --name=syncthing \
  -p 8384:8384 -p 22000:22000/tcp -p 22000:22000/udp \
  -v $HOME/syncthing-config:/var/syncthing/config \
  -v $HOME/syncthing-data:/var/syncthing \
  syncthing/syncthing:2.0.16

# from a release tarball (any OS) — see
# https://github.com/syncthing/syncthing/releases

# verify
syncthing --version    # syncthing v2.0.16
```

## License

MPL-2.0 — see
[LICENSE](https://github.com/syncthing/syncthing/blob/main/LICENSE).
The daemon binary is freely redistributable; MPL-2.0 is file-level
copyleft (modifications to MPL-licensed files must remain MPL, but
linking with other-licensed code is fine).

## One Concrete Example

```bash
# 1. start the daemon — first run generates a self-signed cert and
#    prints the device ID, then opens the web UI on localhost:8384
syncthing serve --no-browser

# 2. in another terminal, look up your device ID
syncthing cli show system | jq -r '.myID'
# ABCDEFG-HIJKLMN-OPQRSTU-VWXYZ12-3456789-ABCDEFG-HIJKLMN-OPQRSTU

# 3. on device B, add device A by ID (web UI: Actions → Add Device)
#    or via the CLI:
syncthing cli config devices add \
  --device-id ABCDEFG-HIJKLMN-OPQRSTU-VWXYZ12-3456789-ABCDEFG-HIJKLMN-OPQRSTU \
  --name laptop

# 4. share a folder — pick a folder ID (any string, must match on
#    both sides) and a local path
syncthing cli config folders add \
  --id work-notes --label "Work Notes" --path ~/notes

# 5. on the other device, accept the folder and pick its local path
#    (web UI prompts; CLI: same command with the folder ID)

# 6. inspect sync state
syncthing cli show pending folders
syncthing cli show connections

# 7. lock down the web UI before exposing to LAN — set a password
#    and turn on TLS in the GUI section of ~/.config/syncthing/config.xml
#    or via:
syncthing cli config gui user set me
syncthing cli config gui password set
```

## Niche It Fills

**Continuous bidirectional file sync with no central server, no
cloud account, end-to-end TLS.** The space is Dropbox /
iCloud / OneDrive / Google Drive (cloud-mediated, vendor sees the
files), [`rclone`](../rclone/) (one-shot or scheduled sync to a
cloud bucket, not continuous bidirectional), [`restic`](../restic/) /
[`kopia`](../kopia/) (backup, not sync — restore is a deliberate
action), `unison` (continuous bidirectional, but stateful single
pair, no NAT traversal, less polished). Syncthing sits in the
corner labelled *"continuous, multi-device, bidirectional, P2P,
self-hosted, with NAT traversal and conflict-copy on collision."*
Pick syncthing when you want Dropbox-style "edit a file on the
laptop, see it on the desktop seconds later" without a third party
seeing the bytes.

## Why use it

Three things `syncthing` does that pay off immediately:

1. **No account, no central server.** The trust model is per-pair:
   you exchange device IDs out-of-band (paste a string into the
   web UI on each side) and the daemons negotiate TLS-mutual-auth
   directly. Discovery and relay are optional public services that
   only see encrypted traffic; you can disable them and run
   LAN-only or via your own relay.
2. **Conflict copies, not silent overwrites.** If two devices edit
   the same file while offline and then reconnect, syncthing
   keeps both — the older one becomes
   `file.sync-conflict-<date>-<id>.ext` next to the winner. You
   never lose work to a sync race.
3. **Versioning is a folder setting.** Per-folder you can enable
   "trash can" / "simple file versioning" / "staggered" /
   "external" — kept versions live in `.stversions/` so a
   recursive `rm` on one device does not vaporize the data on the
   others.

## Weakness

**You run a daemon per device and you manage trust by hand.** No
"share a link, recipient clicks it, file appears" workflow — both
sides must be running syncthing and must have exchanged IDs.
The web UI is the management surface; the desktop tray apps and
mobile clients are community-maintained and vary in polish.
Initial sync of a large folder is slower than rclone copying the
same files to a cloud bucket because syncthing scans + indexes
both sides before transferring.

## When to choose

- You want continuous bidirectional sync between devices you own
  and you do not want a vendor seeing the files.
- You have a NAS / home server that should mirror a laptop folder
  in real time.
- You need conflict copies instead of "last writer wins" — sync
  errors must never silently destroy data.

## When NOT to choose

- You need to share files with someone *outside your trust
  perimeter* via a link — use a cloud service or
  [`magic-wormhole`](https://github.com/magic-wormhole/magic-wormhole)
  for one-shot transfers.
- You need *backup* (point-in-time restore, retention policies,
  encrypted archives) — use [`restic`](../restic/) or
  [`kopia`](../kopia/). Sync is not backup: a deletion on one
  device propagates to all the others.
- You need one-shot or scheduled sync to a cloud bucket
  (S3 / GCS / Backblaze B2) — use [`rclone`](../rclone/).
