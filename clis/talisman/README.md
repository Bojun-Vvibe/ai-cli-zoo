# talisman

## What it does
Pre-commit / pre-push git hook that scans the staged diff (not the whole repo) for credential-shaped strings, high-entropy blobs, suspicious filenames (`.pem`, `id_rsa`, `.pfx`, `.kdbx`), and known token formats (AWS access keys, Slack webhooks, Google API keys, JWT shapes, SSH private-key headers) and refuses the commit / push with a per-file diff highlighting the offending bytes. Installs as a per-repo git hook (`talisman --githook pre-commit`) or as a global hook template via `~/.git-templates`.

## Why it's interesting
Different shape from `gitleaks` / `trufflehog`: those are oriented at scanning a whole history or working tree on demand and are usually wired into CI; `talisman` lives **at the developer's commit boundary** so the secret never enters the local `.git/objects` store in the first place, which is the only point where the leak is still cheap to fix (no `git filter-repo`, no force-push, no rotation). The `.talismanrc` ignore file is per-repo and per-checksum (not per-pattern) so a one-time accepted false positive does not silently widen the allow-list when the file changes.

## Niche category
Security — pre-commit / pre-push secret-leak guard

## Repo
https://github.com/thoughtworks/talisman

## Version pinned
`v1.37.0`

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Per-repo install (current git repo only)
curl -s https://raw.githubusercontent.com/thoughtworks/talisman/main/global_install_scripts/install.bash | bash

# Or as a single static binary
brew install --cask talisman   # community cask
# or download talisman_darwin_arm64 / talisman_linux_amd64 from the v1.37.0 release
```

## Usage examples
```sh
# Wire into the current repo as a pre-push hook
talisman --githook pre-push

# Wire as a pre-commit hook globally for every new clone
talisman --install --hook-type pre-commit

# One-off scan of the working tree without installing the hook
talisman --scan

# Accept a verified false positive (writes a checksum to .talismanrc)
talisman --interactive
```
