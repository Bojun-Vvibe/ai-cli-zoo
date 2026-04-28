# mask

- **Repo:** https://github.com/jacobdeichert/mask
- **Version:** 0.11.7 (2026-01-10)
- **License:** MIT ([LICENSE](https://github.com/jacobdeichert/mask/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install mask` · `cargo install mask` · prebuilt binaries on the GitHub release page (e.g. `mask-0.11.7-aarch64-apple-darwin.zip`) · binary name is `mask`

## What it does

`mask` is a CLI task runner driven by a `maskfile.md` — a plain
Markdown document where each `##` heading is a task and the fenced
code block under it is the script body. The README of your project
literally is the task runner.

- Tasks are declared as Markdown headings with optional positional
  arguments (`## build (target)`) and named flags (`**OPTIONS**`
  sub-section). The body is a fenced code block; the language tag of
  the fence picks the interpreter (`sh`, `bash`, `zsh`, `fish`,
  `python`, `ruby`, `node`, `php`, `lua`, `swift`, plus PowerShell /
  cmd on Windows).
- Discovery is hierarchical: `mask <task>` walks up from `$PWD`
  looking for a `maskfile.md`, so subprojects get their own task
  surface without configuration. `mask --maskfile <path>` overrides;
  `mask --introspect` dumps the parsed AST as JSON for editor
  integrations.
- Subtasks are nested headings (`### build > release`); calling
  `mask build release` runs the nested block. `mask --help` is
  generated from the document and stays in sync because the document
  *is* the source.
- Environment variables, working directory, and required-arg
  validation are declarative inline (`**OPTIONS**` block under the
  heading) — no separate config file, no DSL, just Markdown the
  reader can scan in five seconds.

## Why it's interesting for AI / agent workflows

A `maskfile.md` is the single most LLM-legible task surface that
exists today: agents can `cat maskfile.md`, parse Markdown they were
trained on, and reliably figure out what `mask test`, `mask deploy
staging`, or `mask db migrate` will do without learning a custom DSL.
`mask --introspect` returns the same tree as JSON for tool-using
agents that prefer structured input. The interpreter-per-fence design
means an agent-authored maskfile can mix shell, Python, and Node
tasks in one file without inventing wrappers, which makes it a strong
candidate for the "agent writes a runnable repo task index" pattern.

## When NOT to use it

Skip it when you need a real build graph with file-level dependency
tracking and incremental rebuilds — that's [`just`](../just/) for
recipes-with-deps, or `make` / `bazel` / `ninja` for actual builds.
Skip it if the team standard is already `just` or `npm scripts`;
introducing a fourth task runner has a cost. Skip it for one-off
scripts — a plain `bash` file is shorter than a maskfile with one
task.
