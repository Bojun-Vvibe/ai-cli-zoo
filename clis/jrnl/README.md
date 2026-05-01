# jrnl

> **A command-line journal that stores entries in plain text** —
> one human-readable file per journal (or one encrypted blob with
> AES-256 if you turn on `--encrypt`), entries are timestamped
> Markdown-ish lines parsed back into a queryable log; supports
> multiple named journals (`jrnl work`, `jrnl personal`), tag
> search (`@meeting`), date-range filters (`-from "last week"
> -to yesterday`), full-text search, JSON / Markdown / YAML /
> calendar-heatmap export, and an `$EDITOR` round-trip for the
> longer entries you don't want to type as one shell argument.
> Pinned to **v4.2** (commit
> `41652bee305a48a445b314b5fb2ceb9dde1d91f4`,
> [LICENSE.md](https://github.com/jrnl-org/jrnl/blob/v4.2/LICENSE.md),
> GPL-3.0).

Source: <https://github.com/jrnl-org/jrnl>

## TL;DR

`jrnl` is the *zero-friction* end of the personal-notes
spectrum: `jrnl yesterday at 3pm: shipped the migration; ran
green on second deploy` is one shell line and the entry is
written, indexed, and searchable. No app to launch, no folder
to organise, no proprietary database — the storage is one text
file you can `cat` / `grep` / `git commit` without `jrnl`
installed. Optional encryption keeps the file unreadable on a
shared / lost device. Multiple journals (`work`, `personal`,
`dreams`) keep contexts separated without mounting different
trees.

## Install

```bash
# pipx (recommended on every OS — isolates from system Python)
pipx install jrnl

# Homebrew (macOS / Linux)
brew install jrnl

# pip (if you must)
pip install --user jrnl

# Arch:    pacman -S jrnl
# Nix:     nix-env -iA nixpkgs.jrnl

# verify
jrnl --version    # jrnl 4.2
```

First run prompts for journal location + optional encryption;
config lands at `~/.config/jrnl/jrnl.yaml`.

## License

GPL-3.0 — see
[LICENSE.md](https://github.com/jrnl-org/jrnl/blob/v4.2/LICENSE.md).
Copyleft; redistributing modified `jrnl` requires source. For
personal use the license does not constrain anything.

## One Concrete Example

```bash
# 1. quick entry from one shell line — auto-timestamped to "now"
jrnl Today I finally figured out why the deploy was timing out @work

# 2. backdated entry with a title (first line) + body (rest)
jrnl yesterday at 3pm: Shipped v2.1.
This release moved auth to OIDC and dropped the legacy
session-cookie path. @work @release

# 3. open $EDITOR for a longer entry (no quoting hassle)
jrnl

# 4. read recent entries with rich filters
jrnl -from "last monday" -to today --tags
jrnl @work -n 5                    # last 5 entries tagged @work
jrnl -contains "OIDC" --short      # one-line-per-entry index

# 5. multiple journals — declare in jrnl.yaml, then use as subcommand
jrnl dreams Last night I dreamt I was rewriting the build system in awk
jrnl dreams -from january --format markdown > dreams.md

# 6. export for backup or pipeline
jrnl --format json > all-entries.json
jrnl --format yaml --file out/    # one .md file per entry, mkdocs-ready
jrnl --format calendar -from "1 year ago"   # heatmap of writing days
```

## Why This Over a Markdown File + Git

| ask                                              | answer                               |
| ------------------------------------------------ | ------------------------------------ |
| auto-timestamp every entry                       | `jrnl`                               |
| date-range query without `awk` / `sed`           | `jrnl`                               |
| tag faceting (`@meeting`, `@idea`)               | `jrnl`                               |
| encrypted-at-rest journal on a shared laptop     | `jrnl --encrypt`                     |
| dump for an AI / RAG pipeline                    | `jrnl --format json`                 |
| no-CLI access works too                          | `vim ~/journal.txt` — same file      |
| richer linking / backlinks (Obsidian-shaped)     | use [`obsidian`](https://obsidian.md) instead |

`jrnl` does not try to be a knowledge graph. It is a *log*. If
the workflow is "type a thought, forget about it, find it again
six months from now by date / tag / phrase," `jrnl` is the
shortest path.

## Caveats

- The text-file format is `jrnl`'s own (date header + body
  paragraphs). It's *grep-readable* but not interchangeable
  with arbitrary Markdown out of the box; use
  `--format markdown` to export for static-site generators.
- Encryption is per-journal and uses a passphrase you'll be
  prompted for on every read / write (set `JRNL_PASSPHRASE` in
  the keychain to skip the prompt). Lose the passphrase and the
  journal is gone — there is no recovery.
- The natural-language date parser ("yesterday at 3pm",
  "last monday") is good but not perfect on ambiguous inputs
  ("next sunday" can mean either upcoming sunday depending on
  locale). Prefer ISO dates for important entries.
- Sync is your problem: `jrnl` writes one file, you decide
  whether it lives in iCloud / Dropbox / Syncthing / a private
  git repo. The encrypted-blob mode survives untrusted sync;
  the plaintext mode does not.
- Project pace is steady but not fast (v4.2 in late 2024); the
  format is stable, expect bugfixes more than features.
