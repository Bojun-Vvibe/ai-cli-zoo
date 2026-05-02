# bats

> **Bats-core: a TAP-emitting test framework for Bash where each
> test case is a `@test "description" { … }` block evaluated
> under a fresh subshell with assertion helpers (`run`, `assert_*`,
> `assert_output`, `assert_failure`) so a 200-line shell script
> gets the same `red / green / xfail` reporting and CI integration
> that `pytest` / `jest` / `go test` give every other language.**
> Pinned to **v1.11.0** (as of 2025),
> [LICENSE](https://github.com/bats-core/bats-core/blob/master/LICENSE.md),
> MIT.

Source: <https://github.com/bats-core/bats-core>

## TL;DR

`bats` is what to reach for when the thing under test is *itself*
shell — a `Makefile`, a `justfile` recipe, a release script, a
provisioning bootstrap, a `git` hook, the very wrapper functions
in `~/.zshrc` that ship a team's local toolbelt. Each `@test`
runs in its own subshell so `set -e` failures, `cd`, and `export`
do not leak between cases; the `run cmd args...` helper captures
stdout / stderr / exit code into `$output` / `$status` / `$lines`
for assertions; the companion libraries `bats-assert` /
`bats-support` / `bats-file` / `bats-mock` add the matcher
vocabulary (`assert_output --partial 'foo'`,
`assert_file_exists`, `assert_failure 2`); and `--formatter tap`
/ `--formatter junit` slot directly into the same CI runners that
parse `pytest --junitxml`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install bats-core

# npm (any platform with Node) — the official npm channel
npm install -g bats

# From source (vendor into a repo as a git submodule for hermetic CI)
git clone https://github.com/bats-core/bats-core.git
./bats-core/install.sh /usr/local

# Run
bats test/      # discover *.bats files and run every @test
```

## License

[MIT](https://github.com/bats-core/bats-core/blob/master/LICENSE.md),
SPDX `MIT`.

## Niche / positioning

Pick `bats` over [`shellcheck`](../shellcheck/) /
[`shfmt`](../shfmt/) when the question is *behavioral* not
*static* — shellcheck catches "you forgot to quote `$foo`",
shfmt enforces formatting, bats is the only one that asserts
"this script with these args exits 0 and prints 3 lines";
typically all three run side-by-side in the same CI job. Pick
over hand-rolled `assert.sh` shims when the project crosses ~5
test cases (TAP output + per-test isolation + matcher libraries
become worth the dependency). Pick over rewriting the script
under test in Python or Go just to get a real test framework
when the script is genuinely Unix glue and a rewrite would lose
the "every line is a recognisable shell command" property.
Skip when the codebase has *no* shell (use the language's native
test framework) or when the only "shell" is a 5-line wrapper
that calls one binary (a smoke-test in CI is cheaper).
