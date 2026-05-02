# gallery-dl

> **Command-line image-and-gallery downloader for ~300
> sites** — a Python CLI (`gallery-dl <url>`) that walks
> paginated galleries, user feeds, tag searches, and entire
> account histories on image-hosting + social sites and
> writes the originals to disk with structured filenames,
> per-site metadata sidecars, and resume-on-interrupt state.
> Pinned to **v1.32.0** (released 2026-04-24,
> [LICENSE](https://github.com/mikf/gallery-dl/blob/master/LICENSE),
> GPL-2.0).

Source: <https://github.com/mikf/gallery-dl>
(active development is migrating to
<https://codeberg.org/mikf/gallery-dl> — the GitHub repo
remains the release-tag home for now; track both if pinning
URLs in scripts).

## TL;DR

`gallery-dl` is to image hosts what
[`yt-dlp`](https://github.com/yt-dlp/yt-dlp) is to video
hosts — a per-site extractor framework that knows the
pagination, login, rate-limit, and metadata quirks of each
target so a single `gallery-dl <url>` does the right thing
for ~300 sites: Reddit (subreddits, users, saved), Twitter /
X (timelines, likes, bookmarks), Pixiv, DeviantArt, Flickr,
Tumblr, Imgur, Instagram, Mastodon / Pleroma instances,
ArtStation, Behance, Danbooru / Gelbooru / e621 and the
booru family, Kemono / Coomer, FurAffinity, Newgrounds,
plus image hosts (Catbox, Pomf clones), wallpaper sites,
manga / doujinshi readers, and dozens of niche communities.
Output paths are template-driven (`{category}/{user}/{id}_{filename}`),
each downloaded file gets a JSON sidecar with full metadata
(author, tags, timestamp, source URL, EXIF) when
`-o write-metadata=true`, and `--download-archive` keeps a
SQLite-or-text record of fetched IDs so re-running the same
URL only pulls *new* items — the entire workflow is
idempotent and incremental.

## Install

```bash
# Homebrew (macOS / Linux)
brew install gallery-dl

# pipx (recommended Python install — isolated venv)
pipx install gallery-dl

# pip (any Python 3.8+)
pip install --user gallery-dl

# Pre-built single-file executable (Linux / macOS / Windows)
# Download from https://github.com/mikf/gallery-dl/releases/tag/v1.32.0

# Docker
docker pull ghcr.io/mikf/gallery-dl:1.32.0
```

## When to choose gallery-dl

- Need to back up an artist's Pixiv / DeviantArt / ArtStation
  account, a Reddit user's submissions, a Twitter likes
  archive, or a Tumblr blog before it disappears — the
  extractor catalog covers most of these out of the box.
- Want **structured metadata** alongside the files (author,
  tags, source URL, timestamp) for downstream cataloguing in
  Hydrus / Calibre / a custom DB — `-o write-metadata=true`
  drops a sidecar JSON next to every download.
- Incremental sync matters — `--download-archive` makes
  repeated runs fetch only new items, suitable for cron.
- Authenticated sources (private Pixiv bookmarks, Twitter
  account timeline, NSFW boorus, FurAffinity / DeviantArt
  account-only content) — gallery-dl handles cookies
  (`--cookies cookies.txt` exported from the browser) and
  per-site OAuth flows where required.
- Pair with [`yt-dlp`](https://github.com/yt-dlp/yt-dlp) for
  a complete "archive any URL on the public web" toolbox —
  yt-dlp for video, gallery-dl for image galleries, both
  with the same incremental + metadata-rich philosophy.

## When to pick something else

- The target is a **video** site (YouTube, Vimeo,
  TikTok, Twitch VODs) — that is `yt-dlp`'s job; gallery-dl
  has limited video-extractor coverage and is not optimised
  for it.
- One-off "save this single image" — right-click → Save As
  is faster than configuring a CLI.
- The site is not in the
  [supported-sites list](https://github.com/mikf/gallery-dl/blob/master/docs/supportedsites.md)
  and writing a custom extractor is not on the table — a
  generic recursive downloader (`wget -r`,
  [`httrack`](https://www.httrack.com/)) may get further.
- A polished GUI workflow with thumbnail previews and tag
  editing is needed — Hydrus Network is the right tool
  (and Hydrus can *consume* gallery-dl output, so the two
  pair naturally).

## Caveats

- The supported-sites list churns — sites change layouts
  and APIs constantly, so an extractor that worked last
  month may need a `pip install --upgrade gallery-dl` to
  pick up a fix today. Pin to the latest stable for
  archival runs, not an old version.
- Rate-limit and TOS compliance is the user's
  responsibility — most sites' terms forbid bulk download
  of others' content; use accordingly and configure
  `--sleep` / `--sleep-request` to be a polite client.
- Authenticated sources require fresh cookies — exporting a
  `cookies.txt` from a logged-in browser session is the
  most reliable path (the `cookies-from-browser` shorthand
  works on most browsers but breaks when the browser
  encrypts its cookie store, e.g. recent Chrome on
  macOS / Linux).
- The active-development location is moving from GitHub to
  Codeberg — the GitHub mirror remains current for now
  but watch the project's announcements if pinning to a
  specific repo URL in CI.
