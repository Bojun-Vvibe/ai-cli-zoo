# gomi

> **A `rm` replacement that moves files to the system trash and
> lets you restore them with a fzf-style TUI** — no more
> `rm -rf` regret. Single Go binary, cross-platform (macOS /
> Linux / WSL), per-file metadata so restores land back at the
> exact original path. Pinned to **v1.2.1**
> ([LICENSE](https://github.com/babarot/gomi/blob/master/LICENSE),
> MIT).

Source: <https://github.com/babarot/gomi>

## TL;DR

`gomi` (Japanese for "trash") replaces `rm` with a two-step
delete: files go to `~/.gomi/<timestamp>/` along with a
`history.json` entry that records the original absolute path,
size, deletion time, and a short hash. A second command,
`gomi -B` (browse), opens an inline fzf-style picker over the
trash with live preview of file contents and a one-keystroke
restore that puts the file back at its original location —
parents created if needed, conflicts surfaced as a prompt.

The point is not to replace `rm` for power users in scripts
(use `\rm` or `command rm` for those); it is to make the
*interactive* delete safe by default. You alias `rm=gomi` in
your shell, you stop losing work when you type `rm *.go`
intending `rm *.bak`, and once a week you `gomi --prune` to
purge entries older than N days.

## Repo + version + license

- Repo: <https://github.com/babarot/gomi>
- Latest release: **v1.2.1**
- License: **MIT** —
  <https://github.com/babarot/gomi/blob/master/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `master`
- Language: Go

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install babarot/tap/gomi

# go install
go install github.com/babarot/gomi@latest

# from source
git clone https://github.com/babarot/gomi && cd gomi
make build && sudo install -m 0755 ./gomi /usr/local/bin/

# verify
gomi --version    # v1.2.1
```

## Examples

```bash
# alias once in ~/.zshrc / ~/.bashrc
alias rm=gomi

# delete — files go to ~/.gomi/<timestamp>/, original path stored
gomi notes.md src/old.rs build/

# browse + restore (interactive fzf-style TUI with preview)
gomi -B
# ↑/↓ to move, Tab to multi-select, Enter restores to original path
# / to filter, ? for help, Ctrl-D permanent delete

# list trash contents non-interactively
gomi --list

# prune entries older than 30 days
gomi --prune 30d

# escape hatch when you actually want POSIX rm semantics in a script
\rm -rf node_modules    # bypasses the alias
```

## Why it matters

The XDG trash spec (`~/.local/share/Trash/`) has CLI clients
(`trash-cli`, `rip2`, `rmtrash`) — but they are mostly
"delete + maybe restore via `cd ~/.local/share/Trash/files`".
`gomi` is the rare one that ships an actual *picker* with
preview as a built-in command, so the restore workflow is
zero-context-switch. It also stores the full original absolute
path per file (XDG spec stores it too, but most clients don't
expose a one-key restore over it), which means restoring 50
deleted files from a tree with mixed parents is one keystroke
each, not a manual `mv` per file.

## Comparison

| Tool        | Interactive picker | Original-path restore | Multi-file select | Lang | Cross-platform   |
| ----------- | ------------------ | --------------------- | ----------------- | ---- | ---------------- |
| `gomi`      | yes (built-in TUI) | yes                   | yes (Tab)         | Go   | mac / linux / wsl|
| `rip2`      | no (subcommand)    | yes                   | flag-based        | Rust | mac / linux      |
| `trash-cli` | no                 | partial               | no                | Py   | linux            |
| `rmtrash`   | no                 | no                    | no                | sh   | mac (uses Finder)|
| `rm`        | n/a                | n/a — gone forever    | n/a               | C    | universal        |

## License

- License: **MIT**
- Path in repo: `LICENSE`
- URL: <https://github.com/babarot/gomi/blob/master/LICENSE>
