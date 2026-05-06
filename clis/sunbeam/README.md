# sunbeam

> **sunbeam** — pomdtr/sunbeam, a terminal-native command launcher
> that turns shell scripts (or any program speaking a tiny JSON
> protocol) into a Raycast/Alfred-style searchable list/detail UI.
> Pinned to **v1.0.1**, MIT — license file:
> [LICENSE-MIT](https://github.com/pomdtr/sunbeam/blob/main/LICENSE-MIT).

Source: <https://github.com/pomdtr/sunbeam>

## TL;DR

`sunbeam` is a small Go binary that renders interactive TUI views
(searchable lists, detail panes, forms) and dispatches keybindings
back into the script you wrote. The script's only job is to print
JSON to stdout describing the next view; sunbeam handles the input
loop, fuzzy filter, paging, navigation stack, and rendering. The
result is that a 30-line bash script can become a polished launcher
that:

- lists every open GitHub PR, lets you filter by author, hits Enter
  to open in `$BROWSER`, hits `c` to checkout, hits `r` to request
  review;
- searches your password manager, copies on Enter, edits on `e`;
- browses your bookmarks, your `git stash list`, your `kubectl`
  contexts, your `npm scripts` — anything you can express as
  "list of items, each with a few actions".

The point is **language agnosticism + zero runtime**: any program
that can write JSON to stdout (bash, python, node, deno, jq one-
liners) becomes a launcher extension without linking against a UI
framework. A community "extension" registry exists, but the
canonical workflow is to write your own one-file extensions checked
into your dotfiles repo.

## Install

```bash
# Single static Go binary — releases are at
# https://github.com/pomdtr/sunbeam/releases/tag/v1.0.1

# Homebrew
brew install pomdtr/tap/sunbeam

# Go
go install github.com/pomdtr/sunbeam@v1.0.1

# Pre-built binary
curl -sSL https://github.com/pomdtr/sunbeam/releases/download/v1.0.1/sunbeam_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin sunbeam
```

## Example commands

```bash
# Run the built-in extension catalog browser
sunbeam

# Run a one-off extension by URL or path
sunbeam run ./my-extension.sh

# Pipe a JSON view straight in (for prototyping)
echo '{"type":"list","items":[{"title":"hello"},{"title":"world"}]}' \
  | sunbeam read

# Add a permanent shortcut to your config (~/.config/sunbeam/sunbeam.json)
sunbeam extension install ./pr-launcher.sh --alias pr
sunbeam pr   # now invokable directly
```

A trivial extension — `./hello.sh` — that lists the contents of
`$HOME` and lets Enter cd into the chosen directory:

```bash
#!/usr/bin/env bash
set -e
if [ "$1" = "" ]; then
  jq -nc '{title:"Browse home", description:"List ~ entries"}'
  exit 0
fi
ls -1 "$HOME" | jq -Rn '
  {type:"list", items: [inputs | {title:., actions:[{type:"open", target:("file://" + env.HOME + "/" + .)}]}]}
'
```

## Niche it occupies

**Language-agnostic TUI launcher framework driven by stdout JSON**
— the closest analogues in this catalog approach the same job from
very different angles:

- [`gum`](../gum/) / [`huh`](../huh/) — turn shell scripts into
  *prompts* (one input at a time: confirm, choose, write).
  Orthogonal: gum/huh are linear form widgets, sunbeam is a
  multi-view *navigation stack* with persistent fuzzy search.
- [`fzf`](../fzf/) / [`skim`](../skim/) — fuzzy pickers over a
  single stdin stream. sunbeam is structurally fzf + a view
  router + an action dispatcher; you reach for sunbeam when one
  fzf invocation is no longer enough because the action menu
  varies per-item.
- [`wishlist`](../wishlist/) — SSH directory launcher specifically
  for jumping between hosts. sunbeam covers the same shape
  ("list, pick, act") but for *any* domain.
- [`lf`](../lf/) / [`yazi`](../yazi/) / [`xplr`](../xplr/) —
  terminal file managers with scriptable keybindings. They are
  filesystem-shaped; sunbeam is *generic-list-shaped*. You can
  build a file manager *in* sunbeam in 50 lines, but you would not
  build sunbeam in lf.
- [Raycast](https://www.raycast.com) /
  [Alfred](https://www.alfredapp.com) — the macOS GUI launchers
  this is modelled on. sunbeam is the headless / SSH-friendly
  equivalent: same extension shape, runs over a tmux pane on a
  jumphost.

Pairs cleanly with [`gh`](../gh/) (one-line PR-list extensions),
[`glab`](../glab/), [`jira-cli`](../jira-cli/),
[`jq`](../jq/) (most extensions are `command | jq | sunbeam read`),
and [`gopass`](../gopass/) / [`pizauth`](../pizauth/) for
credential-picker extensions.

Caveats: still a 1.x release with a small community — the
extension protocol has settled but the catalog is much smaller
than Raycast's; no native window-management / global-hotkey story
(it is a terminal app, you launch it from a tmux popup or a
desktop-environment keybind that opens a terminal); JSON-over-
stdout means no streaming for very large lists (everything in the
view must fit in memory before render).

## Citation

- Repo: <https://github.com/pomdtr/sunbeam>
- Latest release: **v1.0.1**
- License: **MIT**
- License file: [LICENSE-MIT](https://github.com/pomdtr/sunbeam/blob/main/LICENSE-MIT)
