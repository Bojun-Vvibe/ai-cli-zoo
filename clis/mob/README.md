# mob

- **Repo**: https://github.com/remotemobprogramming/mob
- **Version**: v5.4.2
- **License**: MIT ([LICENSE](https://github.com/remotemobprogramming/mob/blob/main/LICENSE))

## What it is

A single-binary command-line tool for smooth git handoffs in mob programming and pair programming sessions. Wraps the awkward "create a wip branch, commit everything, push, switch driver, pull, reset back into working tree" dance behind three verbs: `mob start`, `mob next`, `mob done`.

## Why it's in the zoo

Mob/pair programming with raw git means every handoff is "stash, commit with a junk message, push, the next driver pulls, soft-resets, and we hope nothing was missed." Easy to forget a file, easy to push to the wrong branch, easy to lose work when someone's checkout is stale. `mob` standardises the protocol: timer-based driver rotation, an isolated `mob/main` wip branch so the real branch stays clean, automatic squash on `mob done`, and a built-in countdown timer that nudges the next handoff. Works with any remote (GitHub, GitLab, Bitbucket, self-hosted), no daemon, no server.

## Install

```sh
brew install remotemobprogramming/brew/mob
# or
go install github.com/remotemobprogramming/mob/v5@latest
```

## Quick example

```sh
mob start 10        # take the keyboard for 10 minutes
# ... hack ...
mob next            # hand off to the next driver
# ... they hack ...
mob done            # squash the wip branch back onto the feature branch
```
