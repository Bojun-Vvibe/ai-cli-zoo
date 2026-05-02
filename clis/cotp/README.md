# cotp

> **Encrypted TOTP / HOTP authenticator for the terminal.**
> A single Rust binary that stores your 2FA seeds in an
> AES-GCM-encrypted vault under `$XDG_DATA_HOME/cotp/`, then prints
> the rolling 6-digit code on demand or in an interactive TUI.
> Pinned to **v1.9.9**
> ([LICENSE](https://github.com/replydev/cotp/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/replydev/cotp>

## TL;DR

`cotp` is a command-line replacement for Authy / Google Authenticator
on the desktop. It supports TOTP (RFC 6238), HOTP (RFC 4226), Steam
Guard, Yandex, and Motp. Seeds are kept in a single encrypted JSON
vault unlocked with a password (Argon2id KDF + AES-GCM); on every
invocation you type the password once and get either a one-shot
code (`cotp -s github`) or a live-updating TUI list. Import from
Aegis, andOTP, FreeOTP+, Authy, Google Authenticator (QR / JSON),
and 2FAS is built in.

## Install

```bash
# Cargo (any platform with a Rust toolchain)
cargo install cotp

# Homebrew (macOS / Linuxbrew)
brew install cotp

# Arch (AUR)
yay -S cotp

# Pre-built binaries
# https://github.com/replydev/cotp/releases/latest

# verify
cotp --version    # cotp 1.9.9
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/replydev/cotp/blob/main/LICENSE).
Strong copyleft: forks and redistributed builds must stay GPL-3.0.
Fine for personal use and internal tooling; check with legal before
embedding into a closed-source product.

## One Concrete Example

```bash
# First launch creates the vault and asks for a master password
cotp
# > Welcome. Insert your password to create a new database:
# > ********
# (vault written to ~/.local/share/cotp/db.cotp)

# Add a new TOTP seed by hand
cotp add
# Issuer: GitHub
# Label:  alice@example.com
# Secret: JBSWY3DPEHPK3PXP

# Add by scanning a QR code from a screenshot
cotp add --qr-code-image ~/Downloads/github-2fa.png

# One-shot: print the current code for the GitHub entry, no TUI
cotp -s GitHub
# 482913

# Interactive TUI: live-updating list of all entries
cotp
#  GitHub      alice@example.com    482913   ▓▓▓▓▓▓░░░░  18s
#  GitLab      alice@example.com    301847   ▓▓▓▓▓▓▓▓░░  09s
#  AWS root    root@account         992014   ▓▓░░░░░░░░  25s
# j/k move; Enter copies the code to clipboard; / filters; q quits.

# Export the vault (still encrypted) for backup to another machine
cotp export --format cotp --path ~/Backups/cotp-2026-05-02.cotp
```

## Niche It Fills

**The "my 2FA seeds should not live on a phone I can lose, and they
should not live in a SaaS vault someone else can subpoena" gap.**
For developers who already live in the terminal, `cotp` puts the
codes one keystroke away (`cotp -s github`), keeps the vault on
local disk in a known file, and lets you back it up like any other
encrypted blob (`rsync`, `borg`, `restic`). No phone, no cloud sync,
no proprietary format.

## Why use it

1. **Local-only encrypted vault.** Argon2id-derived key, AES-GCM
   ciphertext, single file. You own the storage; back it up
   wherever you back up secrets.
2. **Both one-shot and TUI modes.** `cotp -s <issuer>` is scriptable
   (pipe into `pbcopy` / `xclip` / `wl-copy`); plain `cotp` opens
   the live progress-bar TUI for the "I'm logging into five things
   in a row" case.
3. **Imports the formats you actually have.** Aegis JSON, andOTP,
   FreeOTP+, Authy (with the side-channel extraction trick),
   Google Authenticator QR exports, 2FAS — so migrating off a
   phone-based authenticator is one `cotp import` away.

## Vs Already Cataloged

- **Vs [`gpg-tui`](../gpg-tui/):** `gpg-tui` is a GPG keyring
  browser — it manages signing / encryption keys, not OTP seeds.
  Different secret type, different rotation cadence.
- **Vs [`himalaya`](../himalaya/):** `himalaya` is a terminal mail
  client; orthogonal. They might both unlock with the same
  password manager but solve different problems.
- **Vs a phone authenticator (Aegis / Authy / Google Auth):** Phone
  authenticators are great until the phone dies or you're SSH'd
  into a box and need a code *now*. `cotp` is "the codes are on
  the same machine where I'm typing the login form".

## Caveats

- **Master password = single point of failure.** Forget it and the
  vault is unrecoverable. Print the recovery seed list once and
  store it somewhere physical, or keep a second exported copy in
  a separate password manager.
- **Local vault = local backup discipline.** Unlike Authy cloud
  sync, nothing replicates this for you. If you `rm -rf ~/.local`
  without a backup, the seeds are gone.
- **GPL-3.0.** Don't bundle into a closed-source product without
  legal review. Personal / internal use is fine.
- **No push notifications.** TOTP/HOTP only — `cotp` cannot do
  Duo-style push-approval, because that's a proprietary protocol,
  not an open RFC.
- **Clipboard copy depends on a helper.** Needs `xclip` / `xsel`
  (X11), `wl-copy` (Wayland), or `pbcopy` (macOS) on `PATH` for
  the Enter-to-copy action; otherwise the code is just printed.
