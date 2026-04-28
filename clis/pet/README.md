# pet

- **Repo:** https://github.com/knqyf263/pet
- **Version:** v1.0.1 (2024-12-08)
- **License:** MIT ([LICENSE](https://github.com/knqyf263/pet/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install knqyf263/pet/pet` · `go install github.com/knqyf263/pet@latest` · prebuilt binaries on the GitHub release page · binary name is `pet`

## What it does

`pet` is a small, focused command-line snippet manager — a place to
park the long, half-remembered shell incantations you keep
re-googling.

- Stores snippets as plain TOML in `~/.config/pet/snippet.toml` with
  a `command`, a free-form `description`, and an optional list of
  `tag`s; nothing is hidden inside a binary database.
- `pet new` captures the previous shell command verbatim (via a
  `prev` shell function) so you can save a successful one-liner the
  moment after running it, without retyping or escaping anything.
- `pet search` opens an `fzf`/`peco`-driven picker over the snippet
  list with live preview of the description; the selected command is
  printed to stdout (or executed with `pet exec`), so it slots into
  any shell binding.
- Supports parameterised snippets with `<param=default>` placeholders
  — at search time `pet` prompts for each parameter and substitutes
  it into the chosen command before running.
- Syncs the snippet file to a Gist or a private git repo via
  `pet sync`, which is a thin wrapper that pushes/pulls the TOML —
  meaning multiple machines stay in sync through ordinary git
  history, not a proprietary cloud.

## When to use it

Reach for `pet` when your shell history isn't enough and a
project-level `Makefile` or `justfile` is too much: the
twice-a-month commands you can never remember (long `ffmpeg`
invocations, niche `awk` scripts, AWS CLI tag filters, that one
`find` for stale node_modules). Bind `pet search` to a hotkey
(`Ctrl-S` is common) and it turns into a personal cheatsheet that
lives one keystroke away. The Gist sync is good enough for
single-user multi-machine setups without spinning up extra
infrastructure.

## When NOT to use it

Skip it when the snippets are project-scoped and shared with a team
— commit a `justfile` or [`task`](../task/) `Taskfile.yml` to the
repo so everyone benefits and the commands are reviewed in PRs.
Skip it for snippets that take complex flag combinations you'd
rather express as a real CLI — write a small wrapper script in
`~/bin` instead, so it's discoverable via `$PATH` and tab
completion. Skip it as a notes app: it has no Markdown rendering,
no full-text search beyond fuzzy matching, and no attachments;
reach for a notes tool ([`khoj`](../khoj/),
[`frogmouth`](../frogmouth/) over a notes folder, or a wiki) for
prose. And if all you want is "search my shell history better",
[`atuin`](../atuin/) or [`mcfly`](../mcfly/) are a closer fit.
