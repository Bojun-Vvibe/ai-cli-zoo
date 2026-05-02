# f2

> **A cross-platform command-line tool for batch renaming
> files and directories quickly and safely** — a single Go
> binary by Ayooluwa Isaiah that previews every rename before
> touching the filesystem, supports regex find/replace, EXIF /
> ID3 / MIME variables, sequential numbering, and case-insensitive
> conflict detection that catches collisions Windows would
> silently break on. Pinned to **v2.2.2**
> ([LICENSE](https://github.com/ayoisaiah/f2/blob/master/LICENSE),
> MIT).

Source: <https://github.com/ayoisaiah/f2>

## TL;DR

`f2` is the bulk-rename tool that does what `mv` + `for` +
`sed` + `rename` (the Perl one *or* the util-linux one,
depending on your distro) try to do, but with a **dry-run by
default**, structured conflict reporting, and undo. You give
it a regex find pattern, a replacement template (which can
reference capture groups, EXIF tags from photos, ID3 tags from
audio, file MIME type, modification time, sequential indices,
random tokens, etc.), and `f2` first prints a colored
before/after table — nothing is renamed until you re-run with
`-x` (execute). If two renames would collide (case-only on
Windows / macOS HFS+, identical targets, target overwrites
source), it refuses and tells you which pair conflicts. After
execution it writes a backup file you can `f2 --undo` to
reverse, even across a reboot. Replaces `mmv`, `rename`,
`vimv`, `vidir`, `qmv`, and most one-off shell loops for the
"I have 4000 photos to reorganize by EXIF date" case.

## Install

```bash
# Homebrew (macOS / Linux)
brew install f2

# Go install (always latest)
go install github.com/ayoisaiah/f2/v2/cmd/f2@latest

# Single-binary download (GitHub releases)
curl -L https://github.com/ayoisaiah/f2/releases/download/v2.2.2/f2_2.2.2_darwin_arm64.tar.gz \
  | tar xz && sudo mv f2 /usr/local/bin/

# Debian / Ubuntu
curl -LO https://github.com/ayoisaiah/f2/releases/download/v2.2.2/f2_2.2.2_linux_amd64.deb
sudo dpkg -i f2_2.2.2_linux_amd64.deb
```

## Usage

```bash
# Dry-run: rename "IMG_1234.JPG" -> "vacation_1234.jpg"
f2 -f 'IMG_(\d+)\.JPG' -r 'vacation_$1.jpg'

# Same, but actually execute (note the -x)
f2 -f 'IMG_(\d+)\.JPG' -r 'vacation_$1.jpg' -x

# Reorganize photos by EXIF date taken into year/month folders
f2 -f '.*' -r '{exif.dt.YYYY}/{exif.dt.MM}/{f}{ext}' -R -x

# Sequential numbering with zero padding, starting at 100
f2 -f '.*\.txt' -r 'doc_{%03d}{ext}' --start-num 100 -x

# Undo the last rename batch (uses backup file in $XDG_DATA_HOME/f2)
f2 --undo
```

## Why it's interesting

Bulk rename is one of those problems every Unix user has
solved 10 times with fragile shell scripts, and most existing
tools punt on the hard parts: the Perl `rename` doesn't preview,
the util-linux `rename` doesn't do regex, `mmv` has confusing
syntax, and `vidir` requires you to actually edit the names in
`$EDITOR`. `f2` is the rare tool that gets all of (a) safe by
default with explicit `-x` to commit, (b) structured conflict
detection (case-folding aware so it catches what a naive `mv`
would silently overwrite), (c) persistent undo across processes,
and (d) rich variable templates including EXIF / ID3 / MIME so
photo + music library reorganization don't need a separate tool.
v2.x split the code into a reusable Go library, so you can
embed the rename engine in your own tooling. Skip if your
rename is genuinely one-shot (`mv old new` is shorter), or if
you'd rather edit names in `$EDITOR` (use `vidir` from moreutils).
