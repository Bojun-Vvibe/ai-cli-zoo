# cheat

> **Personal command-line cheatsheets** — `cheat <command>` opens
> a Markdown cheatsheet that *you* (or your team) wrote, with
> embedded shell-variable templating for context-aware values,
> a hierarchical search path so personal sheets override
> community sheets, and `fzf` integration so the picker is one
> keystroke away when you can't remember the cheatsheet name.
> Pinned to **v5.1.0** (SPDX: `MIT`,
> [LICENSE](https://github.com/cheat/cheat/blob/master/LICENSE.txt)).

Source: <https://github.com/cheat/cheat>

## TL;DR

`cheat` is a single Go binary that reads cheatsheets from a
configurable list of directories, lets you write your own as
plain Markdown files (`~/.config/cheat/cheatsheets/personal/<name>.md`),
and renders the requested one to the terminal with optional
syntax highlighting. The killer property is **first-class
personal cheatsheets that override community defaults**: when
the team's `kubectl` cheat conflicts with your personal note,
your personal one wins via the path search order, and a `cheat
-e kubectl` opens the editor on the version that resolves at
the top of the chain.

## Install

```bash
# Homebrew
brew install cheat

# Go install
go install github.com/cheat/cheat/cmd/cheat@latest

# Pre-built binary
curl -LO "https://github.com/cheat/cheat/releases/download/5.1.0/cheat-linux-amd64.gz"
gunzip cheat-linux-amd64.gz && chmod +x cheat-linux-amd64
sudo install cheat-linux-amd64 /usr/local/bin/cheat

# first run: pick the default config; clone community sheets
cheat                              # prompts to create ~/.config/cheat/conf.yml
git clone https://github.com/cheat/cheatsheets ~/.config/cheat/cheatsheets/community

# verify
cheat -v   # cheat 5.1.0
```

## License

MIT — see
[LICENSE](https://github.com/cheat/cheat/blob/master/LICENSE.txt).

## One Concrete Example

```bash
# 1. open a cheatsheet
cheat tar
#   # To extract an uncompressed archive:
#   tar -xvf /path/to/foo.tar
#   # To create a gzipped archive:
#   tar -czvf /path/to/foo.tgz /path/to/foo/
#   ...

# 2. list all cheatsheets resolvable on the search path
cheat -l
#   tag    file                                   title
#   ----   ----                                   -----
#   net    /home/me/.config/cheat/.../curl.md     curl
#   k8s    /home/me/.config/cheat/.../kubectl.md  kubectl
#   ...

# 3. search across cheatsheet bodies (regex over all files)
cheat -s 'http2'
#   curl: --http2          Use HTTP/2
#   nginx: listen 443 ssl http2;
#   ...

# 4. filter by tag
cheat -t k8s -l

# 5. edit (or create) a personal cheatsheet
cheat -e mycommand              # opens $EDITOR on personal/mycommand.md

# 6. fzf-driven picker (bind to a keystroke)
cheat -l | fzf | awk '{print $2}' | xargs cheat
```

A personal cheatsheet with templated variables:

```markdown
% awsprofile, aws

# List S3 buckets in the current profile:
aws --profile=<profile> s3 ls

# Switch profile (defined in ~/.aws/config):
export AWS_PROFILE=<profile>
```

When you run `cheat awsprofile`, `cheat` prompts for
`<profile>` (or honours `--var profile=prod`), substitutes,
and prints the result — so the cheatsheet is parameterised
not just static.

## Niche It Fills

**The personal-cheatsheet loop.** Three workflows it eats:
- "I figured out the right `ffmpeg` invocation last quarter
  and I want it back in 2 seconds, not 20 minutes of
  re-Googling" → `cheat -e ffmpeg` once, `cheat ffmpeg`
  forever after.
- "The team has a shared cheatsheet repo for our internal
  build verbs; everyone's `cheat` resolves them via a
  shared `community` path; my `personal/` overrides for the
  one verb where my workflow differs" — the path search
  order is the team-vs-individual seam.
- "I just learned a new `awk` idiom; I want to write it
  down in a place I'll actually find again" — `cheat -e
  awk` is one keystroke; the editor opens on the existing
  awk sheet, ready for an addition under the right
  heading.

The catalog already has [`tldr`](../tldr/) /
[`tealdeer`](../tealdeer/) / [`tlrc`](../tlrc/) (read-only
community pages) and [`navi`](../navi/) (fzf snippet picker
that copies to clipboard / runs the snippet). `cheat` is
distinct: **personal Markdown cheatsheets you write and own**,
with a hierarchical override chain, view-by-default (not
copy/run by default), and search across the body text not
just the snippet description.

## Why use it

1. **You write the cheatsheets.** The community pack is a
   starting point; the value compounds when you add a new
   sheet every time you solve a non-trivial command. Six
   months in, your `~/.config/cheat/cheatsheets/personal/`
   is the most useful directory on the laptop.
2. **Hierarchical override.** `personal/` shadows
   `community/` shadows `team/` (or whatever order you
   configure) — so the team's `kubectl` cheat is the
   default for everyone, but your personal addition wins
   on your machine without forking the team repo.
3. **Variable templating.** `<profile>` / `<region>` /
   `<bucket>` placeholders prompt at view time (or accept
   `--var name=value`), turning a static reference into a
   parameterised template — closer to a snippet than a
   man page.
4. **Body search.** `cheat -s 'http2'` greps every
   cheatsheet body — when you remember "I wrote down
   something about HTTP/2 somewhere" but not which
   cheatsheet, the search finds it.
5. **Plain Markdown on disk.** Each sheet is a `.md` file
   with an optional `% tag1, tag2` front-matter line. The
   format is `cat`-able from any host without `cheat`
   installed; `git diff` on the sheets directory shows
   exactly what you learned this quarter.
6. **Editor-first ergonomics.** `cheat -e <name>` opens
   `$EDITOR` on the file (creating it if missing), so the
   write loop is one keystroke — no `mkdir -p`, no
   filename guessing.

## Vs Already Cataloged

- **Vs [`tldr`](../tldr/) / [`tealdeer`](../tealdeer/) /
  [`tlrc`](../tlrc/):** Those are *read-only views* of the
  community-curated [tldr-pages](https://tldr.sh) corpus —
  one canonical cheatsheet per command, written by the
  community, no personal overrides, no body search, no
  templating. `cheat` is the *write* surface for the
  cheatsheets *you* wish existed and the *team-shared*
  cheatsheets that are not generic enough for tldr-pages.
  They compose: pick `tldr` for the community baseline,
  `cheat` for the personal layer on top.
- **Vs [`navi`](../navi/):** `navi` is an `fzf`-driven
  snippet picker that copies the chosen snippet to the
  clipboard or executes it (with parameter prompts) —
  optimised for *running* commands. `cheat` is optimised
  for *reading* commands (the default action prints the
  whole sheet so you see context, not just one line). They
  compose: keep one-liner runnable snippets in `navi`,
  multi-step procedural cheatsheets in `cheat`.
- **Vs [`pet`](../pet/):** `pet` is a snippet manager (one
  command + one description per snippet, searchable via
  `fzf`). `cheat` is multi-command Markdown cheatsheets
  per topic — coarser-grained and view-shaped. Pick `pet`
  when the unit is a snippet you want to copy; pick
  `cheat` when the unit is a topic you want to read.
- **Vs `man` pages:** `man` is the upstream-authoritative
  reference. `cheat` is the curated subset *you actually
  use*, in the order *you* think about it, with *your*
  examples — closer to a runbook than a reference.
- **Vs a wiki / Notion / Obsidian:** Those are cross-
  device knowledge bases with rich UI. `cheat` is at the
  shell prompt where the work happens — zero context
  switch, accessible inside `tmux` / SSH / tty1.

## Caveats

- **Personal sheets only matter if you write them.** The
  community pack is a useful baseline, but the value of
  `cheat` compounds with the personal layer. Plan to
  invest 5 minutes/week for the first three months adding
  what you learn — after that the read/write ratio flips
  and the time spent is paid back many times over.
- **No automatic sync.** `~/.config/cheat/cheatsheets/`
  is a local directory; multi-device sync is your job.
  The recommended pattern is `git init` it and push to a
  private remote (or include it in the [`chezmoi`](../chezmoi/) /
  [`yadm`](../yadm/) dotfiles repo).
- **Markdown rendering is plain.** `cheat` ships basic
  syntax highlighting (`chroma`); it does not aspire to
  [`glow`](../glow/)-class rich rendering. If you need
  prose-heavy cheatsheets that render with full Markdown
  fidelity, pipe through `glow`: `cheat foo | glow -`.
- **Path-search order matters.** Misconfiguring `cheatpaths`
  in `conf.yml` (wrong order, missing `tags:`) leads to
  surprising override behaviour. Read the config example
  in `cheat --conf` once before adding the second
  cheatpath.
- **Variables are prompt-time, not edit-time.** A sheet
  with `<profile>` placeholders prompts every view unless
  you pass `--var profile=prod`. For frequently-used
  values, an alias (`alias chap='cheat awsprofile --var
  profile=prod'`) is the right escape hatch.
