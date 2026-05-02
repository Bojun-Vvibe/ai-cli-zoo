# beets

- **Upstream:** https://github.com/beetbox/beets
- **Version:** v2.10.0 (released 2026-04-19)
- **License:** MIT (`LICENSE`)
- **Language:** Python

## What it is

`beets` is a music library manager and tagger. Point it at a directory tree of music files and it
identifies each release against MusicBrainz / Discogs / Spotify metadata sources, rewrites tags
with canonical artist / album / track / year / MBID values, optionally renames + moves the files
into a configurable directory layout (`$albumartist/$album/$track $title.$ext`), embeds + fetches
album art, computes ReplayGain, deduplicates, and stores everything in a queryable SQLite library
that the `beet` CLI exposes through a small but expressive query DSL (`beet ls genre:jazz year:2010..`).
The 2.x line introduced portable libraries (database paths are stored relative to the library root
so the whole collection can be moved without breaking queries) and concurrent metadata-source lookups.

## Why an AI/CLI user might pick it

It is the only mature, scriptable Unix tool that turns "a 30 GB pile of inconsistently-tagged
audio files from 15 years of different rippers" into a database-backed collection that downstream
tools (HTTP servers, embedding pipelines, transcription / classification jobs, dataset builders for
audio ML) can rely on for stable identifiers and consistent paths. Every command emits structured
output (`-f '$path'` / `-f '$mb_albumid'`), the plugin model is just Python files dropped into
`~/.config/beets/beetsplug/` so adding a custom rule is a 20-line script, and the same library
SQLite file is readable by `sqlite-utils` / `datasette` / pandas for ad-hoc analysis. Niche but
unique: where most "media organizer" tools are GUI apps with opaque state, beets is a CLI whose
state is plain files + one SQLite database that survives the next decade of rewrites.

## Install

```sh
pipx install beets
```

## Example

```sh
# Import a directory, auto-tag against MusicBrainz, move files into the configured layout
beet import ~/Downloads/new-rips/

# Query the library: list all FLAC tracks from 2020+ tagged as jazz, by path
beet ls -p format:FLAC genre:jazz year:2020..
```
