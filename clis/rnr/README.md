# rnr

> **rnr** — ismaelgv/rnr, a batch file & directory renamer that runs
> a regex over a list of paths, **previews** every rename as a diff,
> refuses on collisions, and writes a backup file you can `rnr
> --undo` back to the previous state. Pinned to **v0.5.1**, MIT —
> license file:
> [LICENSE](https://github.com/ismaelgv/rnr/blob/master/LICENSE).

Source: <https://github.com/ismaelgv/rnr>

## TL;DR

`rnr '<regex>' '<replacement>' <files...>` is the entire core
contract:

- **Dry-run by default** — without `-f` / `--force`, rnr prints the
  intended renames as a colored diff and changes nothing on disk.
  The "I want to see what it does first" mode is the default, not
  the opt-in flag.
- **Regex with capture groups** — Rust `regex` crate semantics, so
  `rnr '^IMG_(\d+)' 'photo_$1'` works as written. `--no-dump` to
  skip writing the undo file when you really mean it.
- **Collision detection** — if two source paths would map to the
  same target, rnr refuses and prints which inputs collide instead
  of clobbering one with the other. Same for case-fold collisions
  on case-insensitive filesystems (macOS HFS+, Windows NTFS by
  default).
- **Persistent undo** — every actual rename writes a JSON backup
  file to the working directory (or the path passed to `--dump`),
  and `rnr --undo backup.json` reverses every operation in the
  backup. Survives terminal close, machine reboot, and the "wait,
  what did I just do" panic 30 seconds later.
- **Recursive + filterable** — `-r` recurses, `-D` includes hidden
  files, `-t` includes directories (default is files only),
  `--max-depth N` caps recursion, `-x` excludes a regex.

The whole CLI surface is small enough to memorize after one
session, which is why it earns the slot over `mmv` / `rename` /
the Perl `prename` script: those have stranger flag conventions and
none of them have the dry-run-default + undo-file pair.

## Install

```bash
# Single static Rust binary — releases at
# https://github.com/ismaelgv/rnr/releases/tag/v0.5.1

# Cargo
cargo install rnr --version 0.5.1

# Homebrew
brew install rnr

# Arch (community)
pacman -S rnr

# Pre-built tarball
# https://github.com/ismaelgv/rnr/releases/download/v0.5.1/
```

## Example commands

```bash
# Dry-run a regex rename across a directory of photos
rnr '^IMG_(\d+)\.JPG$' 'photo_$1.jpg' *.JPG

# Apply for real (note the -f / --force flag is required to act)
rnr -f '^IMG_(\d+)\.JPG$' 'photo_$1.jpg' *.JPG

# Recursive across a tree, hidden files included
rnr -rfD ' ' '_' some/messy/tree/

# Strip a common prefix from a downloaded folder of audio
rnr -f '^Track \d+ - ' '' downloads/album/*.mp3

# Lowercase every filename in the current dir
rnr -f --replace-limit 0 '([A-Z])' '\L$1' *

# From a list piped on stdin (find / fd / git ls-files)
fd -e log /var/log/myapp | rnr -f --from-file - '\.log$' '.log.old'

# Roll back the last batch (rnr-dump.json was written next to it)
rnr --undo rnr-dump.json
```

## Niche

Batch file & directory renamer — regex-driven, dry-run by default,
collision-safe, with a persistent undo log.

## Why it matters

Bulk rename is a pure shell-fluency tax: every Unix engineer can
write the `for f in *; do mv "$f" "${f// /_}"; done` loop, but the
loop has no preview, no collision check, no undo, and gets long
quickly when the regex needs capture groups. `rename` (Perl) and
the util-linux `rename` overlap in name and *not* in semantics so
half the StackOverflow answers don't apply on your distro. `mmv` is
old, single-author, and missing modern-shell ergonomics. GUI batch
renamers exist (Bulk Rename Utility, Metamorphose, NameChanger) but
none compose into a shell pipeline.

rnr fills the niche of **"regex-shape rename, with the ergonomics
that make it safe to run on 4000 files"** — preview is default,
collision is hard-error, undo is a `--undo` flag away. The verb is
small enough that pairing it with [`fd`](../fd/) for the input list
(`fd ... | rnr -f --from-file - <regex> <repl>`) covers most
scripted bulk-rename cases without writing a one-off Python script.

Pick rnr over [`f2`](../f2/) when the regex-first model fits
better than f2's template-DSL model (f2 is the "rename by EXIF
date / ID3 tag / mtime" tool with a richer per-attribute template
language; rnr is the smaller, regex-only tool — they coexist
cleanly: f2 for media-attribute renames, rnr for everything else).
Pick over [`vidir`](https://github.com/madx/moreutils) (from
moreutils) when the verb is *non-interactive* batch (vidir opens
the file list in `$EDITOR` for hand-editing — the right answer
when a regex won't fit, the wrong answer when it will). Pick over
GNU `mv` and `for f` loops once the rename touches more than ~10
files or needs a dry-run + undo safety net. Pair with
[`fd`](../fd/) (selection), [`fselect`](../fselect/) (SQL-shaped
selection), and [`fclones`](../fclones/) (de-duplicate first, then
rename the survivors with rnr).

Caveats — slow release cadence (v0.5.1, project is feature-stable
and maintained at low velocity; pin the version), the undo file
is per-invocation in CWD by default so a `cd` between rename and
undo loses the file (use `--dump /path/to/keep.json`), the
`--from-file -` stdin mode is the right way to feed a curated
selection but the default no-`<files>`-arg behavior is to operate
on CWD which can surprise — always pass an explicit path glob,
and the regex flavor is Rust `regex` (no PCRE lookaround / no
backrefs in the pattern).

## Verified facts

- Repo: <https://github.com/ismaelgv/rnr>
- Latest release tag: `v0.5.1`
- License: MIT — `LICENSE` at repo root
- Language: Rust
