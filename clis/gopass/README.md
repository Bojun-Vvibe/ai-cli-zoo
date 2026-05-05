# gopass

> **The standard Unix password manager, reimplemented in Go,
> team-aware** — a single-binary CLI that stores every secret as
> a GPG-encrypted file in a git-versioned tree (`~/.local/share/gopass/stores/<name>/`),
> compatible with `pass(1)`'s on-disk layout but extended with
> multi-store mounts, per-store recipient lists for team
> sharing, native YAML/key-value secret bodies, OTP / TOTP
> generation, browser-extension and JSON-API integration, and
> built-in `git push/pull` synchronisation. Pinned to **v1.16.1**
> (commit `f4bb1ded498cb75be25217ee0c784f78b2fb8ba9`,
> [LICENSE](https://github.com/gopasspw/gopass/blob/master/LICENSE),
> MIT).

Source: <https://github.com/gopasspw/gopass>

## TL;DR

`gopass` is what you reach for when `pass(1)` is the right
shape (one encrypted file per secret, git as the sync layer,
GPG as the trust root, your data on disk in a format you can
inspect with `tar` and `gpg` if the binary disappears
tomorrow), but you also need: more than one store mounted at
once, share-with-the-team via a per-store recipient list, TOTP
codes for the same accounts you already have passwords for,
and a JSON API that browser extensions and CI can talk to.
Single Go binary, no Python / Ruby / shell-glue dependency
chain.

## Install

```bash
# Homebrew (macOS / Linux)
brew install gopass

# apt (Debian / Ubuntu)
wget -qO- https://packages.gopass.pw/repos/gopass/gopass-archive-keyring.gpg \
  | sudo tee /usr/share/keyrings/gopass-archive-keyring.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/gopass-archive-keyring.gpg] https://packages.gopass.pw/repos/gopass stable main" \
  | sudo tee /etc/apt/sources.list.d/gopass.list
sudo apt update && sudo apt install gopass

# Go (any OS with a Go toolchain)
go install github.com/gopasspw/gopass@latest

# from a release binary (Linux x86_64 example)
curl -Lo gopass.tar.gz "https://github.com/gopasspw/gopass/releases/download/v1.16.1/gopass-1.16.1-linux-amd64.tar.gz"
tar xf gopass.tar.gz && sudo install gopass /usr/local/bin/

# verify
gopass version    # gopass 1.16.1
```

A working GPG keypair is a prerequisite — `gopass setup` walks
you through pointing it at an existing key or generating one.

## License

MIT — see [LICENSE](https://github.com/gopasspw/gopass/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. one-time setup: pick a GPG key, init the default store
gopass setup

# 2. add a password (prompts hidden, stored as GPG-encrypted file)
gopass insert work/aws/staging-root

# 3. generate a 32-char password and store it (auto-copies to clipboard)
gopass generate -c personal/github 32

# 4. retrieve, copy to clipboard for 45 seconds
gopass show -c work/aws/staging-root

# 5. structured secret (key/value body) — gopass parses YAML-ish syntax
gopass insert -m work/db/prod <<'EOF'
hunter2-the-actual-password
url: postgres://prod.example.internal:5432/app
user: app_ro
EOF
gopass show work/db/prod url           # postgres://...

# 6. TOTP for the same account
gopass otp work/github/2fa             # prints current 6-digit code

# 7. mount a separate team store (shared via git, encrypted to team keys)
gopass mounts add team-ops git@github.com:acme/ops-secrets.git
gopass team-ops/pagerduty/api-key

# 8. add a teammate to a store (re-encrypts every file to the new recipient)
gopass recipients add --store team-ops 0xDEADBEEFCAFE1234

# 9. sync (git pull --rebase + git push, all stores)
gopass sync

# 10. JSON API for browser extension / scripts
gopass api-listen   # speaks the gopass JSON-RPC protocol on stdin/stdout
```

## Niche It Fills

**Team-aware, multi-store evolution of `pass(1)`.** The
encrypted-file-per-secret + git niche has three serious
inhabitants: original [`pass`](https://www.passwordstore.org/)
(Bash + GPG + git, perfect for one machine + one human, awkward
for teams), [`passage`](https://github.com/FiloSottile/passage)
(pass's design but with `age` instead of GPG — modern keys, no
team story), and `gopass` (pass's on-disk format, plus
multi-store, plus team recipient lists, plus TOTP, plus a JSON
API, in one Go binary). Everything else in the password-manager
space is either a hosted service (1Password, Bitwarden), a
sync-via-vendor blob (KeePassXC kdbx files), or a OS-native
keyring (`security`, `secret-tool`) — none of which give you
"every secret is a file my team and I can `git log`".

## Vs Already Cataloged

- **Vs OS keychains (`security` on macOS, `secret-tool` on
  Linux):** keychains are single-machine, single-user, opaque
  blob. `gopass` is multi-machine via git, multi-user via
  recipient lists, and the on-disk format is auditable
  (`tar tf` + `gpg --decrypt` works without the binary).
  Use the keychain for OS-integrated unlock; use gopass for
  the secrets that should follow you across machines and
  teammates.
- **Vs [`age`](../age/) / [`rage`](../rage/):** orthogonal —
  `age` is the file-encryption primitive, `gopass` is a secret
  *manager* (organisation, search, sync, recipient management,
  TOTP, browser bridge) that happens to use GPG as its
  primitive. If you wanted the `age`-on-pass design, you would
  pick [`passage`](https://github.com/FiloSottile/passage),
  not `gopass`.
- **Vs [`sops`](../sops/):** sops is for *config-file* secrets
  inside a repo (encrypt the `db_password:` field of
  `values.yaml`, decrypt at deploy-time). gopass is for
  *human* secrets (the password you'd otherwise type into a
  browser). Different artefact, different audience — many
  teams run both: sops for app config, gopass for ops humans.
- **Vs [`vault`](../vault/) / [`teller`](../teller/):** Vault
  is a centralised secret broker with policy, audit log,
  dynamic secrets, lease renewal — heavyweight, server-side.
  `teller` is a fan-out reader over many backends (Vault,
  AWS SM, gopass, dotenv). gopass is the offline,
  decentralised, file-per-secret backend that one or both of
  those might wrap.

## Caveats

- **GPG is the trust root.** GPG's UX is famously sharp —
  expired subkeys, `gpg-agent` socket lifecycle, smart-card
  unplug, and the YubiKey / pinentry interaction will all
  surface as gopass errors. If you do not already have a
  working GPG setup, expect a 30-minute first-run cost.
- **`git` is the sync layer, with all its conflict semantics.**
  Two machines editing the same secret between `gopass sync`
  runs produces a normal git merge conflict on a binary GPG
  blob, which is unmergeable — the second one to push must
  resolve by picking a side. This is acceptable for human-paced
  edits, painful for "200 machines all rotating credentials in
  a loop".
- **Team-share cost is `O(secrets × recipients)` re-encrypts
  on recipient change.** Adding or removing a teammate
  re-encrypts every file in the store to the new recipient
  list — fine for hundreds of files, slow for tens of
  thousands.
- **Browser-extension + JSON API surface is a separate
  binary.** The `gopass-jsonapi` and the matching browser
  extension are actively maintained but live in sibling repos;
  if you only ever use gopass from the terminal you can ignore
  this, but it is the path you must take for "fill this login
  form from gopass" workflows.
- **Recovery is on you.** No vendor recovery, no "forgot my
  password" link — losing the GPG private key means losing
  every secret. A printed paper backup of the primary key (or
  an offline YubiKey) is the standard mitigation; gopass does
  not provide it for you.
