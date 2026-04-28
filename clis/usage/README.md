# usage

- **Repo:** https://github.com/jdx/usage
- **Version:** v3.2.1 (2026-04-22)
- **License:** MIT ([LICENSE](https://github.com/jdx/usage/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install usage` · `cargo install --locked usage-cli` · `mise use -g usage` · prebuilt binaries on the GitHub release page · binary name is `usage`

## What it does

`usage` is a **spec language and toolkit for command-line interfaces**
written by `@jdx` (the author of `mise`). You write a `usage.kdl`
file (KDL syntax) that declares your CLI's commands, args, flags,
and completion rules — `usage` then generates: shell completion
scripts (bash / zsh / fish / nushell / pwsh), a man page, a markdown
reference, JSON schema for the spec, and a runnable wrapper that
parses argv against the spec and dispatches to a handler script.
The spec format covers the things a hand-rolled bash CLI usually
gets wrong: short / long flag aliases, repeated flags, value enums,
file / dir / number type validation, environment-variable defaults,
mutually-exclusive flag groups, dynamic completion (`complete=
"git branch"`), subcommand trees, and per-subcommand help.

It is the engine behind `mise`'s task system: any project that drops
a `usage.kdl` next to its scripts gets typed completion + `--help`
+ man pages "for free" without rewriting the underlying bash / python
glue. The CLI also includes `usage generate` for scaffolding a spec
from an existing argparse / clap definition, and `usage parse` for
embedding the parser into another script.

## When to pick it / when not to

Reach for `usage` when you have **a pile of project-local scripts
(`./scripts/build`, `./scripts/deploy`, a `Justfile`, a `Makefile`
that has grown CLI ambitions) and you want them to feel like a real
CLI** — with auto-generated `--help`, tab completion that knows
your subcommand tree, and validation that rejects bad flags before
the script body runs. It is also the right answer when you want
your CLI's spec to be **decoupled from its implementation language**
so the same KDL file can drive bash today and a Rust rewrite
tomorrow without users noticing.

Skip it for a single one-shot script (`getopts` is fine). Skip if
you are already on `clap` / `cobra` / `click` and happy — those
generate completions natively and you do not need a second layer.
Skip when your CLI's argument shape is genuinely dynamic at runtime
(plugin-loaded subcommands that the spec cannot enumerate
statically).

## Why it matters in an AI-native workflow

Agents author and modify CLIs constantly: a new `./scripts/eval`
to run a benchmark, a `./scripts/agent` wrapper around an LLM
binary, a `./bin/tools/*` directory of one-shots that get fed
into MCP servers as tool definitions. The same agent then needs
to **discover the surface area of those CLIs** to call them
correctly — `--help` text, flag types, valid enum values, expected
file paths. A KDL spec is a far better source of truth for an LLM
than scraping `--help` output: it is structured, machine-readable
(`usage spec --json`), and stable across whitespace edits. Pair
this with MCP tool definitions and you get one artifact that
documents the CLI for humans, generates completions for shells,
and feeds tool schemas to the model.

## Example invocations

```bash
# Generate shell completions from a usage.kdl
usage generate completion bash > /etc/bash_completion.d/mycli
usage generate completion zsh > ~/.zsh/completions/_mycli
usage generate completion fish > ~/.config/fish/completions/mycli.fish

# Generate a man page and a markdown reference
usage generate manpage > mycli.1
usage generate markdown > docs/cli-reference.md

# Emit the spec as JSON (useful as an MCP tool schema source)
usage spec --json < usage.kdl

# Run a wrapped script: usage parses argv against the spec,
# rejects bad flags, then execs the handler
usage cmd ./scripts/build --target prod --verbose

# Scaffold a spec from an existing clap / argparse CLI
usage generate spec --from clap-help --bin mycli > usage.kdl

# A minimal usage.kdl looks like:
#   name "deploy"
#   bin "deploy"
#   flag "--env" help="target environment" {
#     arg "<env>" choices="staging" "prod"
#   }
#   cmd "rollback" help="revert last deploy" {
#     arg "<release-id>"
#   }
```

## Alternatives in this catalog

- [`mise`](../mise/) — the runtime version manager that ships
  `usage` as its task-spec engine; if you already use mise, you
  are already one `usage.kdl` away from typed task completion.
- [`just`](../just/) — task runner with a `justfile` DSL; pick
  `just` when you want a `make`-shaped runner with no schema, pick
  `usage` when you want a real CLI surface with completion + help.
- [`mask`](../mask/) — markdown-driven task runner; mask is a
  documentation-first runner, usage is a spec-first CLI generator.
- [`pet`](../pet/) — snippet manager for bare commands you do not
  want to formalise; complementary to usage's "promote a script
  to a real CLI" workflow.
