# granted

> **Cloud-credential broker that opens AWS / Azure / GCP consoles in
> isolated browser profiles and hands short-lived creds to your shell.**
> A Go binary (`granted`, plus the `assume` shell helper) that wraps
> AWS SSO / IAM Identity Center, AWS IAM role-chaining, Azure CLI, and
> GCP `gcloud` flows behind one prompt — `assume <profile>` exports
> 1-hour creds into the current shell, and `granted console
> <profile>` opens a Firefox/Chromium "container" tab scoped to that
> account so multiple AWS accounts can be open side-by-side without
> stepping on each other's cookies. Pinned to **v0.39.0** (released
> 2026-03-28, SPDX: `MIT`,
> [LICENCE](https://github.com/fwdcloudsec/granted/blob/main/LICENCE)).

Source: <https://github.com/fwdcloudsec/granted>

## TL;DR

`assume my-prod` reads `~/.aws/config`, walks any SSO / role-chain /
MFA hops it needs, caches the resulting STS credentials in the OS
keychain, and exports `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`
/ `AWS_SESSION_TOKEN` into the current shell. `assume -c my-prod`
instead opens the AWS Console in a fresh isolated browser profile
that is bound to that account only — so opening `my-prod` and
`my-staging` at the same time gives you two independent sessions
in two visually distinct tabs, not one session that keeps
clobbering the other.

## Install

```sh
# macOS / Linuxbrew (recommended)
brew install common-fate/granted/granted

# Or download a release binary
# https://github.com/fwdcloudsec/granted/releases/tag/v0.39.0

# Wire the shell helper (zsh shown; bash/fish are similar)
echo 'alias assume="source assume"' >> ~/.zshrc
exec zsh

# Verify
granted --version    # 0.39.0
```

## License

MIT — unrestricted. Safe to ship to every laptop on the team and
embed in onboarding scripts. The browser-extension half of the UX
ships from the same repo under the same licence.

## Primary use case

You administer 6+ AWS accounts (prod, staging, dev, security,
billing, sandbox) federated through AWS IAM Identity Center, and
you keep accidentally running `terraform apply` against the wrong
account because your shell still has stale creds from the last
`aws sso login`. Replace the whole flow with `assume <profile>`:
creds are scoped to the current shell, time-boxed by STS, and the
prompt clearly shows which account is currently assumed. For
console work, `assume -c <profile>` opens an isolated browser
profile so prod and staging tabs cannot share cookies.

## What it competes with

- **Plain `aws sso login` + `AWS_PROFILE=…`** — works, but every
  shell you open has to redo the SSO dance, console-tab isolation
  is non-existent, and credentials sit in `~/.aws/sso/cache/` as
  files instead of the keychain. `granted` is "the same thing,
  smoothed."
- **[`aws-vault`](../aws-vault/)** — same idea, narrower scope:
  AWS-only, no first-class browser-isolation story, no Azure / GCP
  surface. Pick `aws-vault` if you only care about AWS IAM
  long-lived keys; pick `granted` if you live in IAM Identity
  Center and/or operate across clouds.
- **[`saml2aws`](https://github.com/Versent/saml2aws)** — focused
  on SAML IdPs (Okta, ADFS, Ping) into AWS. Still useful where SSO
  isn't an option, but `granted` covers the modern IAM Identity
  Center flow that most orgs are migrating to.
- **`gcloud auth login` / `az login` directly** — fine for one
  account; gets messy across many. `granted` unifies the prompt
  and the account-switch UX.

## AI-native angle

Coding agents that touch infra (IaC plans, CloudFormation drift
checks, "why is this Lambda failing" loops) need *scoped*,
*short-lived* credentials. `granted` makes that the default:

- **Per-shell creds, not global creds.** An agent invoked in a
  shell that ran `assume read-only-prod` only ever sees read-only
  prod creds. Spawning a sibling agent in another shell with
  `assume sandbox` keeps the two agents fully isolated.
- **Time-boxed by STS.** Every credential expires (default 1 h);
  there is no long-lived secret for an agent to exfiltrate or
  cache to disk. A prompt-injection attack on the agent buys at
  most the remaining session window.
- **Read-only by construction.** Pair `granted` profiles with
  AWS IAM Identity Center permission sets that grant only
  `ReadOnlyAccess` (or a custom audit policy) to give agents a
  "look but don't touch" lane for diagnosis missions, separate
  from the "deploy" lane a human operator uses.
- **Console handoff is structured.** `granted console <profile>`
  prints a deep link that an agent can include in a PR comment so
  a human reviewer lands directly in the right account/region
  with one click — no "which account again?" round trip.

## Caveats

- **Browser-isolation needs the extension.** The Firefox
  containers / Chromium profile UX requires the Granted browser
  extension to be installed once per browser. Without it, console
  opens still work but lose the per-tab account scoping.
- **It is a credential broker, not a permissions story.** Granted
  hands you whatever creds STS / SSO / your role-chain say you
  can have; it does not constrain them further. Least-privilege
  is still IAM's job.
- **Keychain coupling.** Cached creds live in the OS keychain
  (macOS Keychain, Linux Secret Service, Windows Credential
  Manager). On a fresh CI runner with no keychain you want
  `--no-cache` or a different tool.
- **Shell-helper sourcing is required.** The `assume` command has
  to be `source`d into your shell to mutate env vars; running it
  as a child process exports nothing. The install step above
  handles this for zsh — replicate it for your shell of choice.

## Concrete example

```sh
# One-time: wire SSO into ~/.aws/config (granted can scaffold this)
granted sso populate --start-url https://acme.awsapps.com/start \
                     --region us-east-1

# List what's available
granted profile list

# Assume into a shell — exports creds into THIS shell only
assume acme-prod-readonly
aws sts get-caller-identity        # confirms account + role
terraform plan                     # uses the assumed creds

# Open the same account in an isolated browser tab
assume -c acme-prod-readonly

# Drop the creds when done
unassume
```

The shell prompt updates to show the active profile, expiry
clock, and account ID, so "which account am I in" stops being a
question you have to ask.
