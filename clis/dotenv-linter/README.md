# dotenv-linter

> **A `.env` linter that catches the bugs you only discover at
> 3 a.m. when prod can't read `DATABASE_URL`** — duplicate keys,
> leading spaces, unquoted values with `#`, lowercase keys,
> missing `=`, ending without newline, key alphabet drift across
> `.env` / `.env.example` / `.env.production`. One static Rust
> binary (`~3 MB`), zero runtime, no Node / Python / Docker —
> drops into pre-commit, CI, or `lefthook` and rejects the bad
> `.env` *before* it lands. Pinned to **v4.0.0** (SPDX: `MIT`,
> [LICENSE](https://github.com/dotenv-linter/dotenv-linter/blob/master/LICENSE)).

Source: <https://github.com/dotenv-linter/dotenv-linter>

## TL;DR

Static analysis for `.env` files. Runs ~30 built-in checks
(`DuplicatedKey`, `EndingBlankLine`, `ExtraBlankLine`,
`IncorrectDelimiter`, `KeyWithoutValue`, `LeadingCharacter`,
`LowercaseKey`, `QuoteCharacter`, `SpaceCharacter`,
`SubstitutionKey`, `TrailingWhitespace`, `UnorderedKey`,
`ValueWithoutQuotes`, …) plus a *comparison* mode
(`dotenv-linter compare .env .env.example`) that diffs the *key
sets* across env files so a new `STRIPE_KEY` added to `.env`
without a placeholder in `.env.example` is caught at PR time —
the canonical failure mode of every multi-env codebase.

Output is a one-line-per-finding format with `path:line` codes
suitable for editor problem matchers, and `--fix` / `fix`
auto-rewrites the file in place for the safe checks
(`EndingBlankLine`, `ExtraBlankLine`, `LowercaseKey`,
`TrailingWhitespace`, `UnorderedKey`, `QuoteCharacter`).
Schedules: pre-commit hook, GitHub Action
(`dotenv-linter/action-dotenv-linter`), or vanilla CI step.

## Install

```bash
# Homebrew
brew install dotenv-linter

# Cargo
cargo install dotenv-linter --locked

# One-line installer (releases)
curl -sSfL https://raw.githubusercontent.com/dotenv-linter/dotenv-linter/master/install.sh | sh -s

# Docker
docker run --rm -v "$PWD":/app -w /app dotenvlinter/dotenv-linter:v4.0.0
```

## Usage

```bash
# Lint everything reachable from cwd
dotenv-linter

# Lint specific files
dotenv-linter .env .env.production

# Compare key-sets across env files (the killer feature)
dotenv-linter compare .env .env.example .env.production

# Auto-fix the safe checks
dotenv-linter fix

# Disable a noisy check repo-wide
dotenv-linter --skip UnorderedKey

# CI usage — non-zero exit on any finding
dotenv-linter --quiet || exit 1
```

## Pre-commit

```yaml
# .pre-commit-config.yaml
- repo: https://github.com/dotenv-linter/dotenv-linter
  rev: v4.0.0
  hooks:
    - id: dotenv-linter
```

## Why it matters

`.env` files are the single most under-linted artifact in a
modern codebase: shipped to every dev machine, every CI run,
every container, but typically hand-edited with no schema and
no validator. The failure modes are silent — a trailing space
on `DATABASE_URL=` makes `psycopg2` throw at runtime, a `#` in
an unquoted password truncates the secret, a key the developer
added in lowercase by accident is silently ignored by libraries
that uppercase-match. `dotenv-linter` is the first-class
answer: machine-readable, fast (~milliseconds on a 100-key
file), CI-gateable, and *aware of multi-env drift* via the
`compare` subcommand that no general linter (`yamllint`,
`shellcheck`) understands. Pairs with secret scanners
(`gitleaks`, `trufflehog`, `ripsecrets`) — those catch *what
you must not commit*; `dotenv-linter` catches *whether what you
did commit will parse*. Orthogonal to schema-based config
validators (`taplo` for TOML, `yamllint` for YAML) — `.env` has
no schema language of its own, and this fills the gap.
