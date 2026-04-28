# termscp

- **Repo:** https://github.com/veeso/termscp
- **Version:** v1.0.0 (latest stable, April 2026)
- **License:** MIT ([LICENSE](https://github.com/veeso/termscp/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install termscp` · `cargo install --locked termscp` · `pacman -S termscp` · `scoop install termscp` · `.deb` / `.rpm` / static binaries on the GitHub release page · binary name is `termscp`

## What it does

`termscp` is a feature-rich terminal file transfer and explorer
client with a full TUI. It speaks **SCP, SFTP, FTP/FTPS, S3,
WebDAV, SMB, and Kube** (treating Kubernetes pods as a remote
filesystem) through a single dual-pane Norton-Commander-style
interface. You point it at any combination of local + remote, browse
both sides with arrow keys, and copy / move / chmod / symlink / sync
files between them with one or two key presses. Connection profiles,
SSH keys, and bookmarks are stored encrypted in a local config so
you can `termscp -b prod-bastion` and land directly in the right
remote directory. There is also a non-interactive scripting mode
(`termscp -p host -P 22 …`) for one-shot transfers, and a built-in
text editor and pager so you can open and edit a remote file without
ever dropping out of the TUI.

## When to pick it / when not to

Pick `termscp` when you spend real time moving files between a
laptop and one or more remotes and you're tired of stitching `scp`,
`sftp`, `aws s3 cp`, and `rclone` invocations together by hand —
especially when the remote set is heterogeneous (some SSH boxes,
one S3 bucket, one SMB share, one k8s pod). The dual-pane TUI is
much faster than typing fully-qualified paths repeatedly, and the
encrypted bookmark store means you don't paste hostnames and key
paths into shell history. It is also a good "I just need to poke
at one file on prod" tool because it opens, edits, and writes back
in a single session.

Skip it when your workload is **headless, scripted, multi-GB sync**
between two object stores or filesystems — [`rclone`](../rclone/) is
the right tool there, with better parallelism, bandwidth limiting,
checksumming, and a mature `serve` mode. Skip it for **one-shot
single-file copies** in scripts where plain `scp` / `sftp-batch` is
already in your runbook. Skip it if you want a **mountable** remote
filesystem (use `sshfs`, `rclone mount`, or `s3fs`).

## Why it matters in an AI-native workflow

Coding agents frequently need to move a single file — a generated
config, a captured log, a built artifact — from a remote box back
into the working tree, or vice versa. Doing this with raw `scp`
forces the agent to know the exact remote path, the exact key file,
and to spell every flag correctly on the first try; mistakes are
costly because each round trip is a network operation. `termscp`'s
scriptable mode lets the agent name a saved bookmark instead of
re-deriving credentials, and its TUI mode lets a human reviewer
visually confirm "yes, copy *this* file from *this* directory to
*there*" in a way that's much easier to audit than a shell one-liner.
The protocol-agnostic surface (same UX whether the remote is SSH,
S3, or a k8s pod) means the agent doesn't have to branch its tool
selection per environment.

## Example invocations

```bash
# Open the dual-pane TUI starting in $HOME locally, no remote yet
termscp

# Connect to a saved bookmark by name (encrypted credentials)
termscp -b prod-bastion

# Open SFTP on a host directly (will prompt for password / use ssh-agent)
termscp sftp://deploy@bastion.example.com:22/var/log/app/

# Open an S3 bucket as the remote pane
termscp s3://my-bucket@us-east-1/exports/

# Non-interactive: upload a single file to an SFTP host
termscp -P 22 -u deploy -w 'use-ssh-agent' \
  sftp://bastion.example.com:22/srv/app/config.toml \
  ./config.toml

# Save the current connection as a bookmark (interactive prompt)
# Inside the TUI: press 'b' to bookmark, 'B' to recall, 'D' to delete

# Edit a remote file in $EDITOR without leaving the TUI: select + 'o'
```
