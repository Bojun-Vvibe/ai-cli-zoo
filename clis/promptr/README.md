# promptr

> Snapshot date: 2026-04. Upstream: <https://github.com/ferrislucas/promptr>

A Node CLI that reads a prompt file (with [LiquidJS](https://liquidjs.com/)
templating), attaches a list of source files you name on the
command line, sends the bundle to OpenAI, and **applies the model's
response directly to those files on disk**. There is no diff
preview, no approval gate, no agent loop — one prompt, one round
trip, the files mutate in place. You inspect the result with `git
diff` or your favorite git UI.

In a catalog full of agent loops and TUIs, `promptr` is the entry
that takes the opposite stance: keep the human in `git`, not in
the tool. The tool's only jobs are (a) compose a prompt from a
templated file plus the contents of the files you listed, and (b)
write the model's response back over those files. Everything else
— review, undo, retry — is delegated to git.

## 1. Install footprint

- `npm install -g @ifnotnowwhen/promptr` (binary: `promptr`).
- Node 18+. ~10 MB install plus the OpenAI SDK.
- Configuration: a single env var `OPENAI_API_KEY`. Optional
  `PROMPTR_MODEL` overrides the default (defaults to a current
  OpenAI flagship model; pin explicitly with `-m gpt-4o` for
  reproducibility).
- Prompt files: any path; conventionally `*.txt`, `*.md`, or
  `*.liquid`. `_<name>.liquid` files are includable via
  `{% render '_name.liquid' %}`.

## 2. Repo, version, license

- Repo: <https://github.com/ferrislucas/promptr>
- Version checked: **v6.0.7** (released 2024-05-18; default branch
  `main` last touched 2024-12-10 as of the 2026-04 snapshot —
  promptr is in maintenance mode, but the v6.0.7 binary still
  works against current OpenAI models when you pin `-m`).
- License: MIT. License file at the repo root: `LICENSE.md`.
- npm package: `@ifnotnowwhen/promptr`.

## 3. What it actually does

The full invocation shape:

```
promptr -p prompt_file.liquid path/to/file1.js path/to/file2.js
```

1. Read `prompt_file.liquid`, expand all `{% render '_x.liquid' %}`
   includes from the same directory and from
   `~/.promptr/`.
2. For each path argument, read the file's contents and append it
   to the prompt as a fenced block, prefixed with the path so the
   model knows where it came from.
3. POST to OpenAI Chat Completions with a system prompt instructing
   the model to return *the complete revised contents* of each file
   it changed, fenced and prefixed by path.
4. Parse the response, locate each path-prefixed fence, **overwrite
   the file on disk** with the new contents.
5. Exit. No diff is printed; `git status` and `git diff` are the
   review surface.

There is also `-i / --interactive` which keeps the conversation
open across multiple prompts against the same file set, useful for
"now also fix the imports", "now add a docstring" follow-ups.

The Liquid template layer is the actual leverage — a project can
keep a `_project.liquid` include with its conventions:

```liquid
This project uses Node 22, yarn, ES module imports.
Alphabetize methods. No `module.exports` at the bottom of classes.
Single quotes for strings.
```

…and every prompt file in the repo starts with `{% render
'_project.liquid' %}`, so you don't restate the same standards in
every prompt.

## 4. MCP support

None. promptr does not consume or expose Model Context Protocol
servers; the only tool it has is "overwrite the listed files".

## 5. Telemetry

Off. promptr makes no analytics calls of its own. Network egress is
exactly one OpenAI Chat Completions request per `promptr` invocation
(or one per turn in `-i` mode). The model provider sees the
contents of every file you listed and the expanded prompt; no third
party does.

## 6. Where it fits in the zoo

- vs [`aider`](../aider/) — `aider` shows a diff and asks `y/n`
  before writing, owns the git commit, builds a repo-map, and runs
  in a multi-turn REPL. `promptr` writes immediately and treats git
  as the only undo button. Pick `aider` for "I want to chat with
  my repo and approve each edit"; pick `promptr` for "I want this
  prompt + this list of files → done, in one shell command, in a
  Makefile target".
- vs [`mentat`](../mentat/) / [`cline`](../cline/) — same
  diff-preview-and-approve posture as `aider`; same trade-off.
- vs [`gptscript`](../gptscript/) — both treat the prompt as a
  versioned artifact you check into the repo. `gptscript` is a
  declarative `.gpt` file with tools / args / output schemas and
  agent-style sub-script calls; `promptr` is a templated text file
  that mutates a list of files and exits. Pick `gptscript` for
  "shell-script that talks to LLMs and other LLMs"; pick `promptr`
  for "code-mod over these files using this prompt".
- vs [`micro-agent`](../micro-agent/) — both are bounded "do one
  thing and stop"; `micro-agent` halts when a test passes,
  `promptr` halts after one Chat Completions response. Pick
  `micro-agent` when you have a failing test and want the agent to
  iterate against it; pick `promptr` when you have a deterministic
  edit you want applied once with no loop.
- vs [`mods`](../mods/) — `mods` reads stdin and writes stdout;
  `promptr` reads file paths and writes those same files in place.
  `mods | tee` cannot mutate a tree; `promptr` is built for
  mutating a tree.

## 7. When to pick promptr

- You have a **Makefile / CI step / git hook** that needs
  "apply this prompt to these files", non-interactively, and exit
  with a clean status code.
- You want **prompt files in the repo** with shared
  `_project.liquid` includes for project conventions, reviewable in
  PRs alongside the code they shape.
- You are doing a **bulk refactor** ("rename the `Logger` class to
  `Tracer` everywhere it appears in these 30 files, update call
  sites, keep behavior identical") where the right granularity is
  one prompt over a known file list, not a chat.
- You trust git as your safety net and find diff-preview prompts in
  TUIs slow.

## 8. When NOT to pick promptr

- You want a **diff preview before any write** — use
  [`aider`](../aider/), [`mentat`](../mentat/), or
  [`cline`](../cline/).
- You need **non-OpenAI providers** — promptr is OpenAI-only as of
  v6.0.7. For multi-provider, reach for
  [`aider`](../aider/) or [`opencode`](../opencode/).
- You need **multi-step reasoning / tool use** — promptr is single
  round trip, no loop. Use [`opencode`](../opencode/),
  [`claude-code`](../claude-code/), or
  [`openhands`](../openhands/).
- You need **bleeding-edge maintenance** — last release was
  2024-05; the project still works but is not actively developed.
  If that is a hard requirement, pick a maintained alternative from
  this catalog instead.
- The change spans **files you cannot enumerate in advance** —
  promptr requires the file list on the command line; it does not
  discover files. Use [`aider`](../aider/) (with a repo-map) or
  [`opencode`](../opencode/) (with grep/glob tools).
