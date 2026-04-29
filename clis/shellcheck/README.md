# shellcheck

- **Repo:** https://github.com/koalaman/shellcheck
- **Version:** v0.11.0 (latest stable, 2025-09)
- **License:** GPL-3.0 ([LICENSE](https://github.com/koalaman/shellcheck/blob/master/LICENSE))
- **Language:** Haskell
- **Install:** `brew install shellcheck` · `apt install shellcheck` · `dnf install ShellCheck` · `pacman -S shellcheck` · prebuilt binaries on the GitHub releases page · Docker `koalaman/shellcheck`

## What it does

`shellcheck` is a static analysis linter for `sh` / `bash` /
`dash` / `ksh` shell scripts. It parses the script with its own
hand-written shell parser (so it doesn't need a real shell to be
installed), runs hundreds of rules against the AST, and emits
warnings with stable error codes (`SC2086`, `SC2046`,
`SC1090`, …) plus a one-line explanation and, where possible, a
machine-applicable fix. The error codes are the killer feature:
each one has a permanent wiki page
(https://www.shellcheck.net/wiki/SC2086) explaining the bug
class, why it matters, the canonical fix, and when it's a false
positive — so a reviewer can paste a code in a PR comment instead
of re-explaining "always quote your variables" for the hundredth
time. Output formats include human-readable (the default), GCC
(`--format=gcc`, parseable by every editor's quickfix list),
checkstyle / JSON / JSON1 / SARIF (for CI dashboards and GitHub
code-scanning), and diff (`--format=diff`, applies as a patch
with `git apply` to auto-fix what it can). Disabling a check
inline (`# shellcheck disable=SC2086`) or per-file via a
`.shellcheckrc` lets a project carve out its own policy. The
v0.11.0 release tightened parser handling around heredocs,
arrays, and `[[ ]]`, and added several new checks for unsafe
parameter expansion patterns.

## When to pick it / when not to

Pick `shellcheck` whenever you write more than ten lines of
shell. It is the de-facto standard, ships in every distro and CI
image, integrates with every editor (vim/neovim ALE, VS Code,
Emacs flymake, Sublime, Helix LSP via `efm-langserver`), and the
rule corpus encodes a decade of "ways shell will silently
corrupt your data": forgotten quotes, word-splitting traps,
`[ "$x" = "" ]` vs `-z`, `$?` clobbering, `cd` without `||
exit`, `read` without `-r`, `&&` chains where one stage's failure
should abort, etc. Pair it with [`shfmt`](../shfmt/) (formatter)
in pre-commit and 90 % of shell-script bugs disappear before
review.

Prefer it over [`bashate`](https://opendev.org/openstack/bashate)
when you want semantic checks (data-flow, quoting) instead of
just style; over `bash -n` when you want to catch bugs that are
syntactically valid but semantically wrong (the entire reason
this tool exists); over a custom regex linter when you want
something the rest of the world already knows the error codes
for. Skip it for `zsh` / `fish` / `nushell` — `shellcheck`
intentionally targets POSIX-derived shells (`sh`, `bash`, `dash`,
`ksh`, `busybox sh`) and will refuse or mis-analyse zsh-specific
syntax; use a shell-specific linter or your shell's own
`-n`/`zcompile` for those.

## Example

```bash
# Lint a script with GCC-format output (editor quickfix friendly)
shellcheck --format=gcc deploy.sh

# Apply auto-fixes via the diff format
shellcheck --format=diff deploy.sh | git apply

# CI: emit SARIF for code-scanning, fail on warnings or worse
shellcheck --format=sarif --severity=warning **/*.sh > shellcheck.sarif
```
