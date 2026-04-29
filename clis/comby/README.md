# comby

> **A multi-language structural search-and-replace tool that parses
> source by *balanced delimiters* (parens, brackets, braces, quoted
> strings) instead of by regex, so a pattern like
> `if (:[cond]) { :[body] }` matches whole if-blocks across C / Go /
> Java / JS / TS / Rust / Python without ever writing a parser.**
> Pinned to **v1.8.1** (homebrew),
> [LICENSE](https://github.com/comby-tools/comby/blob/master/LICENSE),
> Apache-2.0.

Source: <https://github.com/comby-tools/comby>

## TL;DR

`comby` sits between `sed` (regex, line-oriented, breaks on
nested delimiters) and `ast-grep` (true tree-sitter AST,
per-language grammar). Its trick is *delimiter-aware
template matching*: a `:[hole]` placeholder in a template
matches any balanced subexpression — `:[cond]` inside
`if (:[cond])` greedily consumes one paren-balanced
subexpression, no matter how many nested calls or brackets
sit inside it. Holes can also be typed (`:[id.]` for
identifiers, `:[s.]` for strings, `:[n]` for newlines, `:[~regex]`
for inline regex constraints). One template + one rewrite
template + a language flag (`-matcher .go`) and you have a
batch refactor that respects strings, comments, and nested
delimiters across the whole repo. The same engine drives
search-only mode (`-match-only`), in-place rewrite (`-i`),
diff preview (`-review`), and JSON output (`-json-lines`) for
piping into editors / agents. ~30 languages ship as built-in
matchers (C, C++, Go, Rust, Java, Kotlin, Swift, JS, TS, Python,
Ruby, OCaml, Haskell, Erlang, Elixir, Scala, Clojure, Dart, PHP,
Lisp, JSON, HTML, CSS, SQL, LaTeX, shell, …) and arbitrary
syntaxes are configurable via a delimiter file.

## Install

```bash
# Homebrew (macOS / Linux)
brew install comby            # currently 1.8.1

# Pre-built binaries
# https://github.com/comby-tools/comby/releases

# Verify
comby -version                # 1.8.1
```

## License

Apache-2.0 — see
[LICENSE](https://github.com/comby-tools/comby/blob/master/LICENSE).
Permissive with a patent grant: vendor, fork, ship inside a
commercial product without copyleft obligations.

## One Concrete Example

```bash
# 1. Replace `fmt.Println(x)` with `log.Info().Msg(fmt.Sprint(x))`
#    across a Go repo — but only at the top level, not inside
#    string literals or comments. `:[arg]` matches any balanced
#    subexpression, including `f(g(h()))`.
comby \
  'fmt.Println(:[arg])' \
  'log.Info().Msg(fmt.Sprint(:[arg]))' \
  -matcher .go -i -d ./cmd ./internal

# 2. Search-only mode — find every `if err != nil { return err }`
#    block, emit JSON for an editor / agent to consume.
comby \
  'if err != nil { return :[ret] }' \
  '' \
  -matcher .go -match-only -json-lines -d .

# 3. Review-each-hunk mode — interactive `y`/`n`/`q` per match,
#    like `git add -p` for refactors.
comby \
  'console.log(:[x])' \
  'logger.debug(:[x])' \
  -matcher .ts -review -d src

# 4. Inline regex constraint on a hole — only rewrite calls whose
#    first argument matches /^"[A-Z]/.
comby \
  'log(:[~"[A-Z][^"]*"])' \
  'logger.warn(:[1])' \
  -matcher .py -i -d .

# 5. Multi-file, parallel, with stats.
comby 'TODO(:[who]):' 'FIXME(\1):' -matcher .everything -i -stats -d .
```

## Niche It Fills

**The "structural sed without writing a grammar" tool.** Same
family as [`ast-grep`](../ast-grep/), [`sd`](../sd/),
[`sad`](../sad/), [`fastmod`], [`amber`] — `comby`'s specific
corner is *delimiter-aware templates that work across ~30
languages out of the box without per-language grammar files*.
Pick when you want one tool that handles a Go / TS / Python
mono-repo refactor with the same template syntax, or when the
target language has no tree-sitter grammar but does have
balanced `()` / `{}` / `[]`.

## Why use it

Three things `comby` does that pay off immediately:

1. **Holes consume balanced subexpressions.** `:[arg]` matches
   `x`, `f(g(h()))`, `{a: 1, b: [2, 3]}`, or any other balanced
   chunk — one template handles all of them. Regex chokes on
   the parens; comby does not.
2. **String- and comment-aware.** A `;` inside a string literal
   is not a statement terminator. A `}` inside a `/* */` is not
   a block close. The matcher knows both, per-language, without
   you specifying anything.
3. **Three modes, one binary.** Search (`-match-only`),
   in-place rewrite (`-i`), interactive review (`-review`),
   JSON pipe (`-json-lines`). Drops into agents, drops into
   `xargs`, drops into `git diff`-style review loops.

For an LLM-CLI workflow, `comby -match-only -json-lines` is
the cheapest way to give an agent precise structural matches
across a heterogeneous repo — the JSON includes file path,
byte range, matched substring, and resolved hole values, which
means the agent can rewrite each match individually instead
of guessing at a global sed.

## Vs Already Cataloged

- **Vs [`ast-grep`](../ast-grep/):** `ast-grep` is true
  tree-sitter AST matching with per-language patterns and
  selectors — more precise (knows the difference between a
  function call and a macro invocation, can match by node
  kind), but you write a different pattern per language.
  `comby` uses one delimiter-aware template across all
  languages — less precise (cannot distinguish identifier
  shadowing, type vs term), more portable. Use `ast-grep` when
  the language is supported and precision matters; use `comby`
  for cross-language refactors and for languages without
  tree-sitter support.
- **Vs [`sd`](../sd/):** `sd` is a friendlier `sed` —
  regex-based, line-oriented, no awareness of delimiters or
  strings. Use `sd` for "rename a constant," `comby` for
  "replace a whole if-block."
- **Vs [`sad`](../sad/):** `sad` adds an interactive TUI on top
  of regex find-and-replace. `comby -review` covers the
  interactive use case with structural matching instead of
  regex.
- **Vs [`gron`](../gron/):** `gron` flattens JSON for grep —
  orthogonal. Use `gron` for JSON, `comby` for source code.

## Caveats

- **Not a parser.** No semantic information: `comby` does not
  know that `x.foo()` and `(x).foo()` are the same expression,
  cannot resolve aliases, cannot distinguish a method call from
  a field access in languages that elide parens. For
  type-aware refactors, reach for `ast-grep` or a real
  language-server-driven refactor.
- **Whitespace sensitivity.** Default templates are
  whitespace-sensitive in some matchers — `if(x)` and `if (x)`
  may not both match the same template; use the language flag
  to enable the right tokenizer.
- **Hole syntax is its own DSL.** `:[id]`, `:[id.]`, `:[id\n]`,
  `:[~regex]`, `:[id:e]`, `:[id:n]` — small, but real. Plan to
  re-read the [matchers reference](https://comby.dev/docs/) the
  first few times.
- **Big rewrites are still big rewrites.** Always run with
  `-review` or `-stats` first, then commit before the in-place
  pass; comby has no built-in undo.
