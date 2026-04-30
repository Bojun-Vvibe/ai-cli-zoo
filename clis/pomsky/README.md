# pomsky

- **Repo:** https://github.com/pomsky-lang/pomsky
- **Latest version:** v0.12.0 (release `v0.12.0`)
- **HEAD on `main`:** `06ecdab`
- **License:** Dual MIT / Apache-2.0 — [LICENSE-MIT](https://github.com/pomsky-lang/pomsky/blob/main/LICENSE-MIT), [LICENSE-APACHE](https://github.com/pomsky-lang/pomsky/blob/main/LICENSE-APACHE)
- **Category:** dev productivity / regex authoring

A **portable, readable language that compiles to regular
expressions** for the engine of your choice — PCRE, JavaScript,
Java, .NET, Python, Ruby, Rust. You write a structured Pomsky
expression with named character classes (`Letter`, `Digit`, `Greek`),
explicit repetition (`'http' 's'? "://"`), real comments, real
whitespace, and named capture groups (`:host(['a'-'z' '.']+)`),
then `pomsky` lints it and emits the equivalent flavor-specific
regex string. Compared to raw regex, the source is **diff-readable
and code-reviewable**: a contributor unfamiliar with the project
can guess what `:year([0-9]{4}) '-' :month([0-9]{2})` does without
running it through regex101, and a refactor that moves a group
boundary shows up as a one-line diff instead of a wall of
backslashes. The compiler also catches whole classes of mistakes
at compile time — unbalanced groups, unsupported lookbehind on
the target flavor, ambiguous octal vs back-reference — that a
runtime regex engine would either silently miscompile or only
fail on the unlucky input months later.

## Install

```bash
# macOS
brew install pomsky

# Cargo (any platform with a Rust toolchain)
cargo install pomsky-bin

# Linux / macOS — prebuilt binary from a release
curl -L -o pomsky.tar.gz https://github.com/pomsky-lang/pomsky/releases/download/v0.12.0/pomsky-cli-x86_64-unknown-linux-musl.tar.gz
tar xzf pomsky.tar.gz && sudo mv pomsky /usr/local/bin/

# verify
pomsky --version
```

## Usage examples

```bash
# Compile a Pomsky source to a JavaScript-flavor regex on stdout:
echo "'http' 's'? '://' [w '.' '-']+ ('/' [w '.' '/' '-']*)?" \
  | pomsky --flavor js -
# Output: /http(?:s)?:\/\/[\w.-]+(?:\/[\w./-]*)?/

# Compile a file targeting PCRE (e.g. for a Perl / PHP / nginx regex):
cat > url.pom <<'EOF'
let scheme = 'http' 's'?;
let host   = [w '.' '-']+;
let path   = '/' [w '.' '/' '-']*;
:url(scheme '://' host path?)
EOF
pomsky --flavor pcre url.pom
```

## Why it matters

Regular expressions are the most-written, least-reviewed code in
most repos. A 60-character regex on line 412 of a parser is
effectively a write-once primitive — nobody refactors it, nobody
unit-tests every branch, and the next engineer copies it into a
new file with one tweak that subtly changes the language it
matches. Pomsky moves regex into the same review surface as
ordinary code: the source is the artifact you commit, the
flavor-specific regex is a build output you can regenerate, and
`pomsky test` (with inline `test { match: "https://example.com" }`
blocks) gives you the same red/green TDD loop you already have for
functions. For projects that target multiple languages from one
spec — a URL validator that must match identically in a Go service
and a TypeScript SDK and a PostgreSQL `CHECK` constraint — Pomsky
is the only mainstream tool that lets you **author once and emit
the dialect each engine actually understands**, instead of
maintaining three subtly-divergent regex copies forever.
