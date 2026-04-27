# nushell

> **A new type of shell** — a Rust shell where pipelines
> carry *structured data* (tables, records, lists) instead of
> raw byte streams, with built-in commands for JSON / YAML /
> TOML / CSV / SQLite / parquet and a typed plugin system.
> Pinned to **v0.112.2**
> ([LICENSE](https://github.com/nushell/nushell/blob/main/LICENSE),
> MIT).

Source: <https://github.com/nushell/nushell>

## TL;DR

`nu` rethinks the shell pipeline: instead of `ps | grep node
| awk '{print $2}' | xargs kill`, you write `ps | where name
=~ 'node' | get pid | each { kill $in }`. Each pipeline stage
operates on a typed table, columns are first-class, and
parsing JSON / YAML / CSV / SQLite is a built-in (`open
data.json | where age > 30 | select name email`). It runs as
a login shell or as a one-off interpreter (`nu -c '...'`)
inside any other shell pipeline, and the same syntax works
the same on macOS, Linux, and Windows.

## Install

```bash
# Homebrew (macOS / Linux)
brew install nushell

# Cargo (any OS with a Rust toolchain)
cargo install nu --locked

# winget (Windows)
winget install nushell

# verify
nu --version    # 0.112.2

# one-shot from another shell
nu -c "ls | where size > 1mb | sort-by modified | last 5"

# query JSON without jq
nu -c 'open package.json | get dependencies | columns'

# launch as interactive shell
nu
```

## License

MIT — see
[LICENSE](https://github.com/nushell/nushell/blob/main/LICENSE).
Permissive, fine for scripts shipped in proprietary projects.

## Niche It Fills

**Structured pipelines as a first-class shell primitive.**
POSIX shells were designed for byte streams; everything
structured (JSON, CSV, tabular `kubectl` output) gets
repeatedly parsed and reparsed via `jq` / `awk` / `cut`. `nu`
makes the table the pipeline's atom, so column selection,
filtering, sorting, and aggregation are language built-ins
instead of a chain of single-purpose tools. The `open` command
auto-detects format by extension — `open data.parquet`,
`open db.sqlite`, `open config.toml` — and gives you the same
table interface.

## Why it pairs with coding agents

Agents that emit shell pipelines for data inspection
(`kubectl get pods`, `docker ps`, `gh pr list --json`) waste
tokens on parsing boilerplate. With `nu` available, the agent
can write `gh pr list --json number,title,author | from json |
where author.login == 'me'` in one short pipeline instead of
piping through `jq` with a multi-line filter. The structured-
pipeline model is also more robust to agent mistakes — if the
column doesn't exist, `nu` errors out clearly instead of
silently returning empty `awk` output.

## Vs Already Cataloged

- **Vs traditional `bash` / `zsh`:** `nu` does not replace
  them for everything (login shells, sourcing thousands of
  rc-file scripts, POSIX compatibility) but is dramatically
  better for ad-hoc data wrangling. Many users keep `zsh` as
  login shell and reach for `nu -c '...'` for data tasks.
- **Vs `jq` + `awk` + `cut`:** those are sharp single-
  purpose tools; `nu` is a unified language where the same
  `where` / `select` / `sort-by` / `group-by` works across
  JSON, CSV, SQLite, parquet, and command output.
- **Vs Python one-liners:** `nu` is faster to start and the
  syntax is more pipeline-shaped; Python wins for anything
  needing real libraries.

## Caveats

- **Not POSIX.** Existing `bash`/`zsh` scripts won't run
  unchanged. Use `nu` for new scripts and ad-hoc work, keep
  `bash` for `./configure && make` style legacy.
- **Rapid release cadence with breaking changes.** `nu` is
  pre-1.0 and ships ~monthly with occasional language-level
  breakage. Pin a version in CI; read changelogs before
  upgrading.
- **Plugin ABI is unstable.** Custom plugins compiled against
  v0.110 may need a recompile for v0.112; the core shell is
  the stable surface, third-party plugins are not.
