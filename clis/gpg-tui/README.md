# gpg-tui

> Snapshot date: 2026-04. Upstream: <https://github.com/orhun/gpg-tui>

A terminal UI in Rust for managing GnuPG keys without memorising the
30-flag surface of `gpg --edit-key`. Lists secret + public keys with
expiration, capability, and trust badges; lets you sign, encrypt,
import, export, set trust levels, generate revocation certificates,
and edit subkeys with two-keystroke commands. Acts as the catalog's
reference for "operate the local GPG keyring from a TUI" — the same
slot `lazygit` fills for git or `lazydocker` fills for containers.

## 1. Install

- `brew install gpg-tui` — Homebrew (macOS / Linux)
- `cargo install gpg-tui --locked` — from crates.io
- `pacman -S gpg-tui`, `nix-env -iA nixpkgs.gpg-tui`, `pkg install gpg-tui`
- Static `.tar.gz` / `.deb` / `.rpm` per platform on the
  [releases page](https://github.com/orhun/gpg-tui/releases)
- Requires `gnupg` (>= 2.2.x) on `$PATH`; the TUI shells out to `gpg`
  for every cryptographic operation, so the trust DB stays in
  `~/.gnupg/` and is interoperable with every other GPG client.

## 2. Version pin

**v0.11.2**. Verify with `gpg-tui --version`.

## 3. License

MIT. SPDX: `MIT`. Full text at
[`LICENSE-MIT`](https://github.com/orhun/gpg-tui/blob/master/LICENSE-MIT)
in the repository root.

## 4. What it does

- Live key list with filter (`/`), copy-fingerprint (`y`), detail
  pane (`enter`), and key-ring switching (`tab`) between secret and
  public.
- Inline actions per key: sign (`s`), encrypt to (`e`), set trust
  (`t`), edit (`c`), delete (`d`), export armored (`x`), generate
  revocation certificate (`g r`), refresh from keyserver (`f`).
- Subkey management — add, revoke, change expiry, change usage flags
  — without dropping into the raw `gpg --edit-key` REPL.
- Import from clipboard, file, or keyserver query (`i`).
- Splash card with key algorithm (RSA / Ed25519 / Cv25519), bit
  length, capability set (S/C/E/A), expiration countdown, and
  primary-vs-subkey relationship rendered as a tree.
- Configurable colour palette and key bindings via
  `~/.config/gpg-tui/gpg-tui.toml`; `--style colored | plain` and
  `--armor` for scripting-adjacent flows.

## 5. Install & first use

```bash
brew install gpg-tui
gpg-tui                              # opens on the secret keyring
# inside the TUI:
#   tab        switch secret <-> public
#   /alice     filter keys whose UID matches "alice"
#   enter      open detail view for the highlighted key
#   y          copy fingerprint to clipboard
#   x          export armored public key to a file
#   q          quit
gpg-tui --style colored --splash    # eye-candy mode for screenshots
```

## 6. Example: rotate a subkey before expiry

```bash
gpg-tui
# 1. tab to the secret keyring
# 2. filter to your primary key UID
# 3. enter to open the detail view
# 4. press `c` to edit, navigate to the encryption subkey
# 5. `e` to extend expiry, pick "1y", confirm with passphrase
# 6. `x` to export the refreshed public key, then publish:
gpg --keyserver hkps://keys.openpgp.org --send-keys <FPR>
```

## 7. Why it matters in this catalog

GPG remains the substrate for signed git commits, signed releases
(`gh release create --notes-file ... ` + `git tag -s`), and signed
container images (`cosign` falls back to GPG-style key files in
`offline` mode). When an AI agent needs to verify or sign artifacts
on a developer workstation, the friction is not the cryptography —
it's remembering the seven `--edit-key` subcommands that rotate a
subkey or set ownertrust. `gpg-tui` collapses that ritual to a
keystroke each, which means review-loop instructions can say
"open `gpg-tui`, press `c`, extend expiry to 1y" instead of pasting
a brittle here-doc into the agent prompt.

## 8. Alternatives

- `gpg --edit-key` directly — what `gpg-tui` wraps; always available
  but unfriendly under time pressure.
- [`age`](../age/) / [`rage`](../rage/) — modern file-encryption
  tools that sidestep GPG entirely; pair with `ssh-ed25519` keys for
  recipients. Use when you control both ends and don't need the
  GPG web of trust.
- [`sops`](../sops/) — secret-file manager that consumes GPG (and
  age, KMS, Vault) recipients; `gpg-tui` is how you keep the
  recipient keys themselves tidy.
- Seahorse / Kleopatra — graphical desktop equivalents on
  GNOME / KDE; not headless-friendly.

## 9. Telemetry

No analytics, no outbound network calls of its own. Network traffic
only happens when you explicitly trigger a keyserver action (`f` /
`i` from a server URL), and it goes to whichever keyserver `gpg.conf`
is configured to use. Safe to run on air-gapped machines.
