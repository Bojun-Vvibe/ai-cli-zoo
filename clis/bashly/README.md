# bashly

> **Bash-CLI generator: write a YAML spec, get a self-contained
> Bash script with subcommands, flags, validations, completions,
> and `--help` already wired up.** Pinned to **v1.3.8**
> ([LICENSE](https://github.com/bashly-framework/bashly/blob/master/LICENSE),
> MIT).

Source: <https://github.com/bashly-framework/bashly>

## TL;DR

`bashly` is a Ruby gem (distributed as a Docker image too) that
turns a `bashly.yml` describing your CLI's command tree, flags,
arguments, and validations into a single ready-to-ship `bash`
script. The generated script is plain Bash 3.2+ with no runtime
dependency on Ruby or `bashly` itself — you can hand it to a
sysadmin who has never heard of Ruby. Bashly handles the parts
that are tedious to hand-roll: `--help` rendering, flag /
argument parsing with type coercion, environment-variable
fallbacks, dependency checks (`requires: jq`), config-file
loading, completion scripts (Bash / Zsh / Fish), and a
sub-command dispatch table. Your code lives in `src/` as one
small `*_command.sh` per leaf command, which `bashly generate`
splices in.

## Install

```bash
# Ruby gem (recommended for development)
gem install bashly        # requires Ruby 3.0+

# Docker (no Ruby on host)
alias bashly='docker run --rm -it --user $(id -u):$(id -g) \
    --volume "$PWD:/app" dannyben/bashly'

# Homebrew tap
brew install dannyben/bashly/bashly

# verify
bashly --version    # bashly 1.3.8
```

The Docker image is the canonical "no-install" path used by CI
jobs that need to regenerate a CLI script from `bashly.yml`
without provisioning a Ruby toolchain on the runner.

## License

MIT — see
[LICENSE](https://github.com/bashly-framework/bashly/blob/master/LICENSE).
The generator is MIT, and the **generated** Bash script carries
no Bashly licensing — it is your code under your license.

## One Concrete Example

Build a `db-tool` CLI with two subcommands (`backup`, `restore`),
typed flags, and an `--env-file` fallback, in under two minutes.

```bash
# 1. scaffold
mkdir db-tool && cd db-tool
bashly init
# wrote: src/bashly.yml, src/root_command.sh

# 2. edit src/bashly.yml
cat > src/bashly.yml <<'YAML'
name: db-tool
help: Postgres backup / restore helper.
version: 0.1.0

environment_variables:
  - name: DB_URL
    help: Postgres connection string (overrides --url).

dependencies:
  - pg_dump
  - pg_restore
  - jq

commands:
  - name: backup
    help: Dump the database to a timestamped .sql.gz file.
    flags:
      - long: --url
        short: -u
        arg: url
        help: Postgres URL.
        required: false
      - long: --out-dir
        short: -o
        arg: dir
        default: ./backups
        help: Where to write the dump.
    examples:
      - db-tool backup --url postgres://localhost/app
      - DB_URL=postgres://localhost/app db-tool backup -o /var/backups

  - name: restore
    help: Restore a dump file into the target database.
    args:
      - name: dump_file
        required: true
        help: Path to the .sql.gz file produced by `backup`.
    flags:
      - long: --url
        short: -u
        arg: url
        required: true
        help: Target Postgres URL.
      - long: --drop
        help: DROP existing schema before restoring.
YAML

# 3. generate the executable Bash script + per-command source files
bashly generate
# created: src/backup_command.sh
# created: src/restore_command.sh
# created: ./db-tool         <- the shippable single-file script

# 4. fill in src/backup_command.sh
cat > src/backup_command.sh <<'SH'
url="${args[--url]:-$DB_URL}"
out_dir="${args[--out-dir]}"
[[ -z "$url" ]] && { echo "need --url or DB_URL" >&2; exit 2; }
mkdir -p "$out_dir"
file="$out_dir/dump-$(date -u +%Y%m%dT%H%M%SZ).sql.gz"
pg_dump "$url" | gzip > "$file"
echo "wrote $file"
SH

# 5. regenerate (idempotent — only touches changed files) and run
bashly generate
./db-tool --help                           # auto-rendered help
./db-tool backup --help                    # subcommand help
./db-tool backup --url postgres://localhost/app
./db-tool restore /tmp/dump-20260429.sql.gz --url postgres://localhost/app --drop

# 6. emit completions
./db-tool completions bash > /etc/bash_completion.d/db-tool
./db-tool completions zsh  > ~/.zsh/completions/_db-tool

# 7. ship it — one file
sha256sum ./db-tool
scp ./db-tool prod-host:/usr/local/bin/db-tool
```

The generated script is ~25 KB of pure Bash; no eval-of-curl, no
runtime download, no daemon. It is exactly what you would have
written by hand if you had three free afternoons.

## Niche It Fills

**A typed CLI framework — the kind every modern language has
(`clap` for Rust, `cobra` for Go, `click` for Python) — for
Bash.** Hand-rolled Bash CLIs almost always under-implement the
boring parts: `--help` text drifts from reality, `getopts` only
handles short flags, subcommands are nested case statements,
typed validation is `[[ -n "$x" ]]` sprinkled everywhere, and
completions are an afterthought. Bashly removes the excuse: you
declare the CLI in YAML once, the parser, help renderer, and
completions are generated, and your business logic stays a
small Bash function per command.

## Why use it

1. **Spec is the source of truth.** `bashly.yml` lists every
   subcommand, flag, argument, default, env-var fallback, and
   dependency in one file you can review in a PR. `--help`
   output, completion scripts, and the parser are all derived
   from it, so they cannot drift apart.
2. **Generated output is pure Bash.** No Ruby on the target
   host, no Bashly runtime, no `eval` of remote content. The
   script is auditable line-by-line and runs anywhere `bash`
   3.2+ does (which is "everywhere", including macOS's stock
   `/bin/bash` and BusyBox `ash` with minor caveats).
3. **Idempotent regeneration preserves your code.** `bashly
   generate` only writes the dispatcher and per-command
   skeletons that don't yet exist; it never overwrites a
   `*_command.sh` you've edited. The workflow is "edit YAML →
   regenerate → fill in any new command files → commit", and
   `git diff` clearly separates parser changes from logic
   changes.

For LLM-driven workflows, Bashly turns "give me a CLI for X"
into a structured generation problem: produce a `bashly.yml`,
run `bashly generate`, fill in N short Bash functions. Each
function has one clear input (the parsed `${args[...]}` /
`${flags[...]}` map) and one clear output (stdout + exit
code), which is much easier for an agent to write correctly
than a free-form Bash script.

## Vs Already Cataloged

- **Vs [`mask`](../mask/):** `mask` runs commands defined in a
  `maskfile.md` (literate-Markdown task runner — a `make`
  alternative). It's about *invoking* tasks within a project.
  Bashly is about *generating* a standalone CLI binary you
  hand to other people. Different artefact, different lifecycle.
- **Vs [`gum`](../gum/) / [`huh`](../huh/):** those are
  interactive-prompt toolkits — you call them inside your Bash
  script to render menus / inputs. They compose *with* Bashly,
  not against it: a Bashly-generated `db-tool restore`
  command can call `gum confirm` before dropping the schema.
- **Vs hand-rolling with `getopts` / `argbash`:** `getopts`
  only handles short flags and one level of subcommand;
  `argbash` is a closer competitor (also generates Bash from a
  spec) but its spec lives in C-preprocessor-style comments
  inside the script and it does not handle subcommand trees,
  completions, or environment-variable fallbacks as cleanly.
- **Vs writing the CLI in Go / Rust / Python:** if you want a
  real binary with real types, do that. Bashly is for the
  case where the CLI is glue around other shell tools (`pg_*`,
  `aws`, `kubectl`, `jq`, `git`) and rewriting the glue in a
  compiled language adds more friction than it removes.

## Caveats

- **You are still writing Bash.** Bashly generates the parser,
  not your business logic. Quoting, `set -euo pipefail`,
  `IFS` handling, and array semantics are still on you.
  Bashly's `strict: true` toggle adds `set -euo pipefail` to
  the generated header but cannot make Bash a memory-safe
  language.
- **Bash version skew is real.** macOS ships Bash 3.2 (2007)
  as `/bin/bash` for licensing reasons; modern Bash (5.x) is
  installable via Homebrew but lives at `/opt/homebrew/bin/bash`.
  Bashly targets 3.2-compatible output by default, but if you
  use associative arrays or `${var^^}` in your command files,
  the generated script will fail on stock macOS Bash. Test
  with `/bin/bash ./your-cli` before shipping to mixed audiences.
- **Generated script size grows with the command tree.** A
  CLI with 50 subcommands generates a ~200 KB script — still
  trivially shippable, but `git diff` on the generated file
  becomes unreviewable. The convention is to commit `src/`
  and the generated script, and to read PRs by diffing
  `src/bashly.yml` and `src/*_command.sh`, not the artefact.
- **Ruby toolchain for development.** The generator needs Ruby
  3.0+ (or the Docker image). For a CI pipeline that only
  needs to regenerate, prefer the Docker image — it pins the
  Bashly version exactly and avoids Ruby version drift across
  developer machines.
