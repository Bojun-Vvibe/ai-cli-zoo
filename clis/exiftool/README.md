# exiftool

> **Read, write, and edit metadata in just about any file format
> ever invented** — a single Perl script (and shipped as a
> standalone binary) that handles EXIF, IPTC, XMP, ICC, GPS,
> Maker Notes, ID3, PDF info, video container metadata, and
> hundreds of other tag groups across photos, video, audio,
> documents, and raw files. Pinned to **13.57** (released
> 2025-10-31,
> [LICENSE](https://github.com/exiftool/exiftool/blob/master/LICENSE),
> Artistic-1.0-Perl OR GPL-1.0-or-later).

Source: <https://github.com/exiftool/exiftool>
(canonical home: <https://exiftool.org>)

## TL;DR

`exiftool` is the Swiss-army-knife metadata reader/writer that
every photo workflow eventually depends on. It understands more
file formats and more tag dictionaries than any other open-source
tool — Phil Harvey's tag tables are effectively the de-facto
reference for camera Maker Notes, and software like Lightroom,
darktable, digiKam, and immich either embed it or rely on its
output. It can rename files by capture date, copy GPS from a
GPX track onto a folder of photos, strip privacy-leaking tags
before publishing, batch-rewrite copyright notices, or just
dump every tag a file has so you can see what your phone is
secretly recording.

## Install

```bash
# macOS / Linux
brew install exiftool

# Debian / Ubuntu
sudo apt install libimage-exiftool-perl

# Fedora
sudo dnf install perl-Image-ExifTool

# Windows
# winget install OliverBetz.ExifTool
# (or download the Windows EXE from exiftool.org)

# verify
exiftool -ver    # 13.57
```

## Examples

```bash
# dump every tag in a file (pretty-printed, grouped)
exiftool -G -a -s photo.jpg

# read just GPS + capture time
exiftool -GPSLatitude -GPSLongitude -DateTimeOriginal photo.jpg

# strip ALL metadata (privacy scrub before sharing)
exiftool -all= -overwrite_original photo.jpg

# batch-rename a folder of photos to YYYYMMDD_HHMMSS.jpg from EXIF
exiftool '-FileName<DateTimeOriginal' -d '%Y%m%d_%H%M%S%%-c.%%le' ./photos

# geotag a folder of photos from a GPX track recorded by your phone
exiftool -geotag track.gpx ./photos

# set copyright + author across a directory tree, recursive
exiftool -r -overwrite_original \
  -Copyright="(c) 2026 Jane Doe" \
  -Artist="Jane Doe" \
  ./portfolio

# JSON output for piping into jq / scripts
exiftool -json -GPSPosition -DateTimeOriginal *.jpg | jq '.[0]'
```

## Use when

- Any photo / video / audio metadata task that is not "open one
  file in a GUI". CLI batch operations against thousands of
  files are exactly what `exiftool` is built for.
- You need to scrub metadata before publishing — original
  capture device, serial number, GPS, software fingerprints all
  leak out by default and `exiftool -all=` is the most reliable
  one-liner to remove them.
- You are building an asset pipeline (immich, photoprism, a
  custom DAM) and need a deterministic CLI to extract the same
  fields the indexers do.
- You need format coverage no other tool has — obscure raw
  formats, video containers, PDF / EPS / SVG metadata, even
  Lightroom catalog sidecars (.xmp).

Skip `exiftool` for high-throughput bulk *image processing*
(resize, encode) — pair it with `imagemagick` / `vips` /
`ffmpeg` for the pixel work and let `exiftool` handle the tags
afterwards. For pure EXIF reads in a hot path, the Go
`go-exif` or Rust `kamadak-exif` libraries are faster, but
nothing matches `exiftool`'s coverage breadth or its
write-side correctness.
