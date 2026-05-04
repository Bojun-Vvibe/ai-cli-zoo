# aws-vault

> **Secure storage and shell-session injection for AWS credentials.**
> A single Go binary (`aws-vault`) that stores long-lived IAM access
> keys in the OS keychain (macOS Keychain, Windows Credential
> Manager, GNOME Keyring, KDE Wallet, `pass`, or an encrypted file),
> then mints short-lived STS session / role-assumed credentials on
> demand and injects them into a child process's environment. Pinned
> to **v7.2.0** (released 2024-02-21, SPDX: `MIT`,
> [LICENSE](https://github.com/99designs/aws-vault/blob/master/LICENSE)).

Source: <https://github.com/99designs/aws-vault>

## TL;DR

`aws-vault exec prod -- terraform plan` unlocks the keychain entry
for the `prod` profile, calls `sts:GetSessionToken` (or
`sts:AssumeRole` if the profile has a `role_arn`), exports
`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN`
into the environment of the child `terraform` process, and lets
those credentials expire (default 1 hour) when the child exits.
Long-lived keys never touch `~/.aws/credentials` in plaintext, never
sit in your shell history, and never leak through process listings.

## Install

```sh
# macOS:
brew install --cask aws-vault    # or: brew install aws-vault

# Linux:
# Download the static binary from the v7.2.0 release page.
curl -L -o aws-vault \
  https://github.com/99designs/aws-vault/releases/download/v7.2.0/aws-vault-linux-amd64
chmod +x aws-vault && sudo mv aws-vault /usr/local/bin/

# Verify:
aws-vault --version    # v7.2.0
```

Then:

```sh
aws-vault add prod                 # paste keys once, into the keychain
aws-vault exec prod -- aws s3 ls   # short-lived creds, scoped to one command
```

## License

MIT — unrestricted. Safe to ship as part of an internal developer
bootstrap script, bake into a workstation image, or wrap inside a
larger team-tooling CLI.

## Primary use case

A consultant or platform engineer with five-plus AWS accounts (a
prod, a staging, a couple of customer accounts, a personal sandbox)
who needs MFA-gated role assumption and absolutely cannot leave
long-lived `AKIA...` keys in `~/.aws/credentials`. `aws-vault`
keeps the long-lived keys behind the OS keychain unlock prompt,
prompts for the MFA token only when an STS session expires, and
caches the resulting short-lived session so a `terraform plan` /
`terraform apply` / `aws s3 sync` chain doesn't re-prompt for every
command.

## What it competes with

- **Plain `~/.aws/credentials` + `aws configure`** — the AWS CLI's
  default. Long-lived keys sit on disk in plaintext, no MFA story,
  no session-token caching. `aws-vault` is what you reach for the
  first time you have to rotate a leaked key.
- **`aws sso login` + IAM Identity Center** — the official modern
  replacement when your org runs IAM Identity Center (formerly
  AWS SSO). If you have it, use it; `aws-vault` is for the very
  common case where you still have classic IAM users with access
  keys (third-party accounts, legacy orgs, customer accounts you
  don't control).
- **[`granted`](https://granted.dev/)** — newer Common Fate tool
  with a similar shape plus a browser extension for console SSO.
  Heavier (depends on Common Fate's hosted services for some
  features); `aws-vault` is fully local and dependency-free.
- **`aws configure sso` profile + `aws-sso-util`** — a thinner
  wrapper around AWS SSO. Doesn't help with classic IAM users.
- **[`teller`](../teller/) / [`sops`](../sops/) / [`vault`](../vault/)**
  — general-purpose secrets managers. They can store the AWS
  access key, but none of them mint short-lived STS session
  tokens or handle MFA prompting. Pair them with `aws-vault`
  rather than replacing it.

## Why include

The catalog already lists `vault` (HashiCorp Vault), `sops`,
`teller`, `chezmoi`, and `gpg-tui` — generic secret management.
None of them address the specific AWS-IAM short-lived-credential
workflow. `aws-vault` is the de-facto answer in 2026 for engineers
who still juggle classic IAM access keys across multiple accounts
and need MFA-gated role assumption without writing the
`sts:AssumeRole` boilerplate themselves. Worth including as the
canonical AWS-credential-hygiene CLI alongside the more general
secret-manager entries.
