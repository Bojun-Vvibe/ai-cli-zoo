# xc

> **Markdown-as-task-runner** — a zero-config Go CLI that treats
> the `## Tasks` section of your `README.md` (or any markdown
> file) as the task definition: each `### task-name` heading is a
> runnable target, the fenced code block underneath is the script,
> and a `Requires:` / `RunDeps:` / `Dir:` / `Env:` list above the
> fence declares dependencies and environment. Pinned to **v0.9.0**
> (released 2026-01-05, [LICENSE](https://github.com/joerdav/xc/blob/main/LICENSE),
> MIT).

Source: <https://github.com/joerdav/xc>

## TL;DR

`xc` (pronounced "execute") is a Make / Just / Task replacement
whose task file *is* your README. You write a `## Tasks` section
in `README.md` with `### build`, `### test`, `### lint` headings,
each followed by a fenced bash block; `xc build` runs the block.
The killer property is that the same prose your contributors
already read to learn the project ("how do I build this?") is
also literally what executes — no Makefile that drifts from the
docs, no `justfile` that nobody discovers, no `package.json`
scripts buried three levels deep. `xc` (no args) lists every
task with its first paragraph as the description; `xc -md` round-
trips the file to make sure it parses; `xc --help` is just the
README's prose.

Inter-task dependencies are declared as `Requires: build, lint`
on a line above the fence, environment as `Env: GOFLAGS=-trimpath`,
working directory as `Dir: ./web`. The dependency graph is
topologically sorted before execution; failures abort the chain.

## Why it's interesting

Three decades of build-tool churn — Make, Rake, Grunt, Gulp,
npm scripts, Just, Task, Mage, Earthly — have all asked
contributors to learn yet another DSL while leaving the README
to drift out of sync with the actual incantations. `xc` inverts
the relationship: the documentation **is** the runner. New
contributors who read `README.md` to understand the project
have, by the time they finish, also learned every command they
can run, because the commands *are* what they read. Drift is
structurally impossible: if the README says `xc build` runs
`go build ./...`, then `xc build` runs `go build ./...` —
because that block is what `xc` parses and executes.

## Install

```bash
# macOS / Linux
brew install xc

# Go
go install github.com/joerdav/xc/cmd/xc@latest

# verify
xc --version    # 0.9.0
```

## Examples

Add this to your `README.md`:

````markdown
## Tasks

### build

Compile the binary into `./bin/myapp`.

```bash
go build -o bin/myapp ./cmd/myapp
```

### test

Run the full test suite.

Requires: build

```bash
go test ./...
```

### release

Tag and push.

Requires: test
Env: GOFLAGS=-trimpath

```bash
git tag "v$(cat VERSION)"
git push --tags
```
````

Then:

```bash
# list every task xc can find with its prose description
xc

# run one task (deps run first)
xc test            # runs build, then test

# run several
xc build test lint

# show what xc parsed (debugging task discovery)
xc -md
```

## Use when

- Your project already has a "How to build" section in the
  README and you are tired of the README + Makefile drifting
  out of sync.
- You want new contributors to learn the runnable surface by
  reading the docs they were going to read anyway.
- You want a runner with no separate config file at all — the
  README is the config.
- You like Just's ergonomics but dislike that nobody discovers
  the `justfile` exists.

Skip `xc` when your build is genuinely complex enough to need
real artifact-graph thinking (use [`task`](../task/),
[`just`](../just/), Make, or Earthly), when tasks need to be
shared across many repos via a library mechanism (xc is
deliberately per-repo), or when your team would rather have a
machine-checked DSL than prose-with-fences (xc trusts the
markdown — typo a heading and the task is silently invisible).
