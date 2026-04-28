# jq

> **The `sed`/`awk` of JSON** — a single ~1 MB C binary that
> takes a JSON stream on stdin, runs a tiny functional-language
> program over it, and writes JSON back to stdout. Pinned to
> **v1.8.1** (commit
> `4467af7068b1bcd7f882defff6e7ea674c5357f4`,
> [COPYING](https://github.com/jqlang/jq/blob/master/COPYING),
> MIT).

Source: <https://github.com/jqlang/jq>

## TL;DR

`jq` is the universal "I just got JSON back from `curl` / the AWS
CLI / `kubectl` / `gh`, now what" tool. Its language is built
around *filters* that are composed with `|` — `.foo`, `.[]`,
`map(...)`, `select(...)`, `group_by(...)` — and every filter
operates on a stream of JSON values, so the same one-liner works
on a single object, a list, or NDJSON. v1.7 introduced SQL-style
operators (`IN`, `INDEX`, `GROUP_BY`) and proper module support;
v1.8.1 (the pin here) ships further bug fixes around `--stream`,
the regex engine, and the binding scope rules. The project was
dormant 2018–2022, was revived under the `jqlang` GitHub org in
2023, and is now actively maintained again.

## Install

```bash
# Homebrew (macOS / Linux)
brew install jq

# Linux distro packages
# Debian / Ubuntu: apt install jq
# Fedora:         dnf install jq
# Arch:           pacman -S jq
# Alpine:         apk add jq

# Static binary from a release (any glibc / musl Linux, macOS, Windows)
curl -L -o jq \
  https://github.com/jqlang/jq/releases/download/jq-1.8.1/jq-linux-amd64
chmod +x jq && sudo mv jq /usr/local/bin/

# verify
jq --version    # jq-1.8.1
```

## Representative examples

```bash
# 1. Extract a nested field from `gh` output
gh pr list --json number,title,author -L 5 \
  | jq -r '.[] | "#\(.number) [\(.author.login)] \(.title)"'

# 2. Group + count log lines (NDJSON) by status, sort descending
cat access.ndjson \
  | jq -s 'group_by(.status)
           | map({status: .[0].status, n: length})
           | sort_by(-.n)'

# 3. In-place edit a config file (jq itself can't write back, so
#    use the standard sponge-or-tmp dance)
tmp=$(mktemp) && jq '.servers[0].port = 8443' config.json > "$tmp" \
  && mv "$tmp" config.json
```

## When to use vs. alternatives

- Pick **jq** when the input is JSON, the transform is one shell
  command long, and you want a tool that has been on every
  production box since 2014.
- Pick [`yq`](../yq/) (the Go one by mikefarah) when the input is
  YAML or you want jq syntax against TOML / XML too.
- Pick [`fx`](../fx/) when you want to *explore* a large JSON
  document interactively (TUI tree view, search, save filter) —
  `jq` is for writing filters, `fx` is for discovering them.
- Pick [`llm-jq`](../llm-jq/) when you know what shape you want
  but cannot remember the jq syntax — it asks an LLM to author
  the filter and then runs it locally.
- Pick a real programming language (`python -c`, `node -e`) when
  the transform spans more than ~3 lines of jq, has loops with
  state, or needs HTTP calls in the middle.
