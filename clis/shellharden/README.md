# shellharden

- **Repo:** https://github.com/anordal/shellharden
- **Version:** v4.3.1 (latest tagged release, 2023-05; stable since)
- **License:** MPL-2.0 ([LICENSE](https://github.com/anordal/shellharden/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install shellharden` · `cargo install --locked shellharden`
  · `apt install shellharden` (Debian/Ubuntu) · prebuilt binaries on the
  GitHub release page

## What it does

`shellharden` is an *opinionated rewriter* for bash scripts: it
takes a `.sh` file (or stdin) and emits a version of the same
script with every variable expansion and every command substitution
correctly *quoted*, in-place, automatically. Where
[`shellcheck`](../shellcheck/) is a linter that *tells* you `SC2086:
Double quote to prevent globbing and word splitting`, shellharden
goes the next step and *applies the fix*: `cp $src $dst` becomes
`cp "$src" "$dst"`, `for f in $(ls *.log)` becomes the safe glob
form, `rm -rf $TMPDIR/$NAME` becomes `rm -rf "$TMPDIR/$NAME"`. The
default mode is `--check` (exit non-zero if anything would change,
print a unified diff — perfect for CI gating); `--transform`
rewrites stdin → stdout; `--replace` does the in-place edit. Also
folds in a small set of *style* normalizations the bash community
has converged on — preferring `$(...)` over backticks, removing
the `function` keyword in front of POSIX-shaped function
definitions, normalizing `[ ... ]` use cases. The whole tool is
one ~3 MB static Rust binary with zero runtime dependencies. It
ships a curated tutorial,
[*how to do things safely in bash*](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md),
that is widely cited as a reference on quoting, IFS, glob safety,
and signal handling — the fixer encodes the lessons in that
document.

## When to pick it / when not to

Pick `shellharden` whenever you have a bash script — first-party
or vendored — that has not been quoting-audited in a while and you
want the audit to *finish*, not just to leave 47 `SC2086` warnings
in the lint output forever. Drop `shellharden --check **/*.sh` into
CI alongside [`shellcheck`](../shellcheck/) and the two tools form
a complete pre-commit gate: shellcheck catches semantic bugs and
anti-patterns (race conditions, missing `set -e`, wrong use of
`[[`), shellharden enforces that the surface is bulletproof against
filenames-with-spaces / variables-with-globs / command-injection-
via-unquoted-substitution. Pair with [`stylua`](../stylua/) /
[`dprint`](../dprint/) / [`taplo`](../taplo/) / [`shfmt`] for the
formatter half of the polyglot pre-commit picture; on the
[`lefthook`](../lefthook/) side, this is a
`stage_fixed: true` candidate so the rewrite gets re-staged
automatically. Particularly valuable for repos full of
operations / deploy / install scripts where filenames may legally
contain spaces (macOS user-supplied paths, S3 keys, Windows
share imports), where one unquoted `$path` is a latent
`rm -rf /home/user/Application Support` waiting for someone to
test on a non-ASCII setup.

Skip `shellharden` on **non-bash shells** — `zsh`, `fish`,
`nushell`, POSIX `dash` scripts that intentionally avoid bashisms.
The parser is bash-shaped; running it on `zsh` features
(`${(@)var}`, `(N)` glob qualifiers, `=()` process substitution
extensions) will at best refuse to touch them and at worst
mis-rewrite. Skip it as your **only** quality gate — it does not
catch logic bugs (typo'd variable names, missing `||` /
`&&` discipline, wrong test operator) or `set -euo pipefail`
hygiene; for those, `shellcheck` is the right neighbour. Skip it
on scripts where you *intentionally* want word splitting (a
deliberately-unquoted `$@` rest-args pass-through, an array-as-
flat-string trick) — read the diff before accepting; the tool
sometimes converts a deliberate idiom into the safer form, which
may not be what the script meant. And note the project is in
maintenance mode (last release 2023; the rewriter has been
stable for years) — that is fine for a deterministic syntactic
fixer, but do not expect new bash-5.x feature support to land on
a regular cadence.

## Example invocations

```bash
# Show what would change in every script under bin/, exit non-zero on diff
shellharden --check bin/*.sh

# Rewrite a single file in place (commit the diff afterwards)
shellharden --replace deploy/release.sh

# Pipe mode — read stdin, write rewritten version to stdout
cat tools/install.sh | shellharden --transform - > tools/install.fixed.sh

# Preview the unified diff for one file without touching it
shellharden --check ops/restart.sh; echo "exit=$?"
# (use --check --suggest to print only the diff hunks, no exit-code dance)

# CI gate — run on every push, fail the build if any script regressed
shellharden --check $(git ls-files '*.sh' '*.bash')

# Pre-commit hook (lefthook example, save as lefthook.yml entry)
#   shellharden:
#     glob: "*.{sh,bash}"
#     run: shellharden --replace {staged_files}
#     stage_fixed: true

# Pair with shellcheck on the same files for the full picture
for f in $(git ls-files '*.sh'); do
  shellcheck "$f" || exit 1
  shellharden --check "$f" || exit 1
done
```
