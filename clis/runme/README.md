# runme

> **Run shell snippets straight out of Markdown** — turn
> any `README.md` / `RUNBOOK.md` / `ONBOARDING.md` into a
> live runbook where every fenced ```` ```bash ```` block
> is an executable step with named ID, retries, env
> propagation, working-directory anchoring, dependency
> ordering, and per-block timeout — single static Go binary
> that also ships an LSP server, a TUI runner, and a VS
> Code extension — pinned to **v3.16.11** (commit
> [`8c108b6`](https://github.com/runmedev/runme/commit/8c108b6424d1e71e950da4992e80e2d484d2cfda),
> [LICENSE](https://github.com/runmedev/runme/blob/v3.16.11/LICENSE),
> Apache-2.0).

Source: <https://github.com/runmedev/runme>

## TL;DR

`make` for your README. The bash blocks in your repo's
docs become first-class targets:
`runme run setup-db && runme run start-api`. The block
metadata (`{name=setup-db cwd=./db env=DATABASE_URL
timeout=120s}`) is parsed from the fenced-block info string,
so the source of truth is still a Markdown file that
renders correctly on GitHub — but it now also *runs*.

The killer property is **single source of truth for docs
and automation**. The same paragraph that explains "to
seed the dev database, run `psql -f seed.sql`" also *is*
the seed step. Stop maintaining `README.md` plus a
parallel `Makefile` plus a `scripts/` directory plus a
`.github/workflows/onboard.yml` that all drift apart
within a quarter — runme makes the doc the executable.

## Install

```bash
# Homebrew (macOS / Linux)
brew install runme

# prebuilt binary (releases)
curl -sSfL \
  https://downloads.runme.dev/runme/3.16.11/runme_linux_x86_64.tar.gz \
  | tar -xz -C ~/.local/bin runme

# Go install
go install github.com/runmedev/runme@v3.16.11

# Debian / Ubuntu
curl -L -o /tmp/runme.deb \
  https://downloads.runme.dev/runme/3.16.11/runme_linux_x86_64.deb
sudo dpkg -i /tmp/runme.deb

# verify
runme --version    # 3.16.11
```

VS Code extension: install `stateful.runme` from the
marketplace and any Markdown file becomes an interactive
runbook with per-block "Run" buttons inline in the editor.

## Example usage

Authoring side — a `RUNBOOK.md`:

````markdown
# Onboarding

## 1. Bootstrap the database

```bash {name=db-up cwd=./infra}
docker compose up -d postgres redis
```

## 2. Apply migrations

```bash {name=migrate cwd=./api timeout=120s}
go run ./cmd/migrate up
```

## 3. Seed dev data (depends on migrate)

```bash {name=seed cwd=./api}
go run ./cmd/seed --env=dev
```

## 4. Start the API

```bash {name=api background=true cwd=./api}
go run ./cmd/api --port=8080
```
````

Operator side — running individual steps or a chain:

```bash
# list all named blocks in the current directory
runme list

# run one named block
runme run db-up

# run several, in order, halting on first failure
runme run db-up migrate seed api

# interactive TUI: arrow-key + Enter through the runbook
runme tui

# run blocks from a different file
runme run --filename docs/RELEASE.md cut-tag

# dry-run: show the resolved command + env without executing
runme print db-up

# expose blocks over the language server (for VS Code, etc.)
runme server
```

Block metadata supported in the info string:
`name`, `cwd`, `env`, `background`, `interactive`,
`promptEnv`, `excludeFromRunAll`, `timeout`, `category`,
`closeTerminalOnSuccess`, `terminalRows`. Variables
exported by one block (via `export FOO=bar`) propagate to
subsequent blocks in the same `run` invocation.

## Why it matters

- **Doc and automation in one file.** The single hardest
  part of any onboarding doc is keeping it current. When
  the same Markdown block both *explains* what to do and
  *is* what is executed, drift becomes structurally
  impossible — a stale block fails loudly the next time
  someone runs it.
- **Renders correctly on GitHub.** The metadata lives in
  the fenced-block info string (`{name=...}`), which
  GitHub's Markdown renderer ignores gracefully. The
  rendered README on the repo page looks like a normal
  README; the same file fed to `runme` is an executable
  runbook. No special directives, no preprocessor.
- **Beats `make` for narrative workflows.** Makefiles are
  excellent for build graphs but hostile for onboarding
  ("what does `make setup` actually do?"). runme keeps
  the *prose* — the why, the prerequisites, the gotchas —
  alongside the command, which is what a new contributor
  actually needs to read.
- **Beats `just` for narrative workflows.** [`just`](../just/)
  is a fantastic command runner with a clean DSL, but its
  recipes live in a `justfile` separate from any docs.
  runme inverts the relationship — the doc *is* the
  recipe file. Pick `just` for a tight set of project
  commands; pick runme for a narrative onboarding /
  runbook / incident-response playbook where the prose
  matters as much as the command.
- **TUI + LSP + VS Code.** The same engine powers the
  CLI runner, a per-block "Run" button in VS Code, and
  an interactive TUI for production runbooks where an
  on-call engineer arrow-keys through a documented
  recovery procedure step by step.
- **Block dependencies and env propagation.** A `seed`
  block declared after `migrate` runs in order; env vars
  exported in earlier blocks (`export DB_URL=...`) are
  visible to later blocks in the same `runme run` call,
  matching the natural flow of a tutorial.

## Vs Already Cataloged

- **Vs [`just`](../just/):** `just` is a clean modern
  `make` replacement with a dedicated `justfile` syntax —
  the right answer for "I have 30 project commands and
  want one short alias each." runme is the right answer
  for "I have a 12-step onboarding doc and want each
  step to be both readable and executable from the same
  file." Different problems; many projects use both.
- **Vs [`mask`](../mask/):** mask is the closest cataloged
  peer — a `Makefile`-style task runner backed by a
  Markdown file (`maskfile.md`). The differences:
  mask treats the Markdown file as a structured task
  manifest with strict heading-based command discovery;
  runme treats arbitrary Markdown documentation files
  with named bash blocks anywhere inside them. Pick
  mask when you want a single dedicated task file with
  Markdown niceties; pick runme when you want every
  README / RUNBOOK / ARCHITECTURE doc in your repo to
  also be runnable.
- **Vs [`mdformat`](../mdformat/) / [`mdsh`](https://github.com/bashup/mdsh):**
  mdformat is purely a formatter; mdsh is the original
  "Markdown as shell script" hack but is bash-only and
  unmaintained. runme is the modern, multi-language,
  metadata-aware successor with first-class IDE
  integration.
- **Vs [`task`](../task/):** Taskfile.dev's `task` is a
  YAML-based task runner with a strong dependency graph
  and parallel execution. Better than runme for build
  pipelines and CI orchestration; worse than runme for
  human-narrative onboarding documents. Use `task` for
  release pipelines, runme for the README that explains
  how a new contributor sets up the dev environment.
- **Vs [`gum`](../gum/) / [`vhs`](../vhs/) for interactive
  scripting:** gum builds interactive prompts, vhs records
  terminal demos — both compose well with runme blocks.
  A runme block can call `gum confirm` for interactive
  approval gates and a vhs cassette can demo a runme
  runbook end-to-end.
- **Vs Jupyter / `nbdev`:** Jupyter is the canonical
  prose-plus-execution surface for data-science
  workflows. runme is the same idea for shell-driven
  ops / devops / onboarding workflows — Markdown
  instead of `.ipynb`, plain text instead of JSON,
  line-orientable diffs in `git`.

## License

Apache-2.0 — see
[LICENSE](https://github.com/runmedev/runme/blob/v3.16.11/LICENSE).
Patent grant included; safe to embed in commercial
products and CI infrastructure. The Stateful platform
(hosted runbook collaboration) is a separate
commercial offering by the same maintainers; the runme
binary itself is fully open source under Apache-2.0.

## Caveats

- **Block-name discipline matters.** Blocks without a
  `{name=...}` are addressable only by Markdown heading
  context, which is brittle if headings get renamed.
  Convention: every executable block gets an explicit
  `name=...` so the runbook stays stable across
  documentation refactors.
- **Trust the file before running it.** `runme run`
  executes shell — never run an unaudited runbook from
  an untrusted repo. Runme is a doc-driven *executor*,
  not a sandbox; treat the Markdown blocks the same way
  you would treat a `Makefile` or `package.json` script.
- **Cross-shell block portability is your problem.**
  Bash blocks behave like bash; PowerShell blocks
  behave like PowerShell. A block targeted at multiple
  platforms still needs to be platform-portable in its
  own commands — runme is not a shell-translation layer.
- **`background=true` blocks need supervision.** Marking
  a block as background (e.g., `api`) starts the
  process and returns immediately; the operator is
  responsible for terminating it. Combine with
  [`hivemind`](../hivemind/) or [`overmind`](../overmind/)
  for a real Procfile-style supervisor when running a
  long-lived dev stack.
- **Active 3.x line.** API surface (block metadata
  keys, CLI subcommands) is stable but evolving — pin
  to a specific version (`go install
  github.com/runmedev/runme@v3.16.11`) in shared
  dotfiles and CI to avoid block-metadata drift across
  machines.

## As of

2026-05-04. Upstream tag `v3.16.11` (2026-04-30). Active
3.x line on a roughly weekly release cadence. The
"named bash blocks in arbitrary Markdown" core model and
the metadata schema have been stable across the 3.x
line; pin to `v3.16.11` for reproducible runbook
behavior across teammates' machines.
