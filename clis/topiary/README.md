# topiary

> **One formatter, many languages, via tree-sitter** — a universal
> code formatter that does not ship a per-language formatter; it
> ships an *engine* that takes a tree-sitter grammar plus a small
> declarative `.scm` query file describing your formatting rules
> ("indent here, soft-break there, no space before `;`") and
> formats any language with a tree-sitter parser. Built and used
> by the Nickel team to format Nickel itself, but ships
> ready-to-use queries for OCaml, OCaml interface files, Bash,
> JSON, TOML, CSS, Tree-sitter Query, Rust (experimental), and
> Nickel. Pinned to **v0.7.3** (commit
> `75ce8324ebaef45e00a964f110ed18ca3ed80235`,
> [LICENSE](https://github.com/topiary/topiary/blob/main/LICENSE), MIT).

Source: <https://github.com/topiary/topiary>

## TL;DR

Most languages either have a blessed formatter (`gofmt`, `rustfmt`,
`black`, `prettier`) or none. For the "none" case you are stuck
hand-rolling style, fighting whitespace in code review, and writing
your own AST-printer if you ever want consistency. `topiary` flips
the model: instead of writing a printer in Rust per language, you
declare your formatting in tree-sitter's existing `.scm` query
language — capture nodes with predicates like `@append_hardline`,
`@prepend_space`, `@indent.begin` / `@indent.end`,
`@delete` (for redundant trailing commas), `@scope_begin.line` —
and the engine deterministically emits formatted output. The
result: adding a new formatted language is "find a tree-sitter
grammar + write a query file", not "build a printer". The CLI
itself is small: `topiary format file.ncl`,
`topiary format --check src/`, `topiary visualise file.ncl` (dump
the tree-sitter parse for query authors). It also exposes a
library API so editors and language servers can format on save
without spawning a process.

## Install

```bash
# Cargo
cargo install topiary-cli --locked

# Nix flake (the recommended path; pulls grammars + queries pinned)
nix run github:topiary/topiary -- format file.ncl

# From source
git clone https://github.com/topiary/topiary
cd topiary
cargo build --release
./target/release/topiary --version    # topiary 0.7.3

# verify
topiary --version
```

Format a file:

```bash
topiary format src/main.ncl
topiary format --check src/      # exit 1 if anything would change (CI mode)
topiary format - < input.json > output.json   # stdin/stdout
```

Add a brand-new language by dropping a tree-sitter grammar and a
`languages/<lang>.scm` query file into the configured query path
(`TOPIARY_LANGUAGE_DIR` or the `[[language]]` table in
`languages.toml`). No code changes to topiary itself.

## Why it's worth a slot in the zoo

There is a long tail of languages — config DSLs, internal IDLs,
small functional languages, query languages, build files — that
will never have a hand-written formatter because nobody has the
time. `topiary` makes "consistent formatting" a configuration
problem rather than a software-engineering problem: write a 200-line
`.scm` file, get a deterministic formatter usable from CLI, editor,
and CI. Even for languages that *do* have a formatter, `topiary` is
interesting because the rules are inspectable and patchable — you
do not need to fork the formatter to disagree with one of its
choices, you edit the query. It is the most "lisp-like" of the
modern formatter ecosystem (the rules *are* the data) and a
genuinely new design point next to `prettier`-style monoliths.

## Where it sits

- vs `prettier` / `gofmt` / `rustfmt` / `black` / `dprint`:
  language-specific formatters with hand-written printers. They
  are faster on a single language and ship more opinionated rules
  out of the box. `topiary` wins when (a) your language has no
  formatter or (b) you want your formatter rules to be data, not
  code.
- vs [`treefmt`](https://github.com/numtide/treefmt): a *meta*
  runner that dispatches to many existing formatters. Often used
  *with* `topiary` — `treefmt` calls `gofmt` for `.go`, `prettier`
  for `.ts`, and `topiary` for everything else.
- vs writing a `tree-sitter`-based printer by hand: `topiary` is
  the abstraction layer; you are not the first person to need
  "indent inside `{ … }`, hardline after `;`, space around `=`".
- vs editor "format on save" with ad-hoc rules: `topiary` rules
  travel with the repo (committed `.scm` files), so CI and every
  contributor see the same output.

## Footguns

- The `.scm` query language is powerful but unforgiving: a
  missing `@scope_end.line` to balance a `@scope_begin.line` will
  produce *plausibly-formatted* output that is subtly wrong.
  `topiary visualise` is the debugger — use it.
- Formatting is non-trivially expensive: tree-sitter parse +
  query evaluation + idempotency check (the engine formats twice
  to confirm fixed-point). Expect ~5–50 ms per file. CI on huge
  monorepos should batch.
- Built-in language coverage is small (OCaml, Nickel, Bash, JSON,
  TOML, CSS, tree-sitter Query, Rust experimental). For Python,
  TypeScript, Go, etc., reach for the language-native formatter —
  do not try to reimplement `black` in `.scm`.
- Idempotency is checked at runtime, not statically. If your
  query is non-idempotent (e.g. inserts a trailing newline that
  the next pass interprets as a new node), `topiary` errors out
  rather than producing oscillating output. Read the error and
  add a balancing rule.
- The Rust query support is explicitly experimental and *not* a
  `rustfmt` replacement; it exists as a stress test of the engine.
- Configuration discovery (`languages.toml`, `TOPIARY_CONFIG_FILE`,
  `TOPIARY_LANGUAGE_DIR`) is layered; if a query "doesn't apply",
  run `topiary config` to see which paths were merged.
