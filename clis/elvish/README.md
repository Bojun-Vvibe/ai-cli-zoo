# elvish

> **A statically-typed, expression-oriented shell with first-class
> data structures** — a single Go binary that replaces `bash`/`zsh`
> with a language where pipelines carry typed values (lists, maps,
> closures), not just byte streams. Pinned to **v0.21.0**
> ([LICENSE](https://github.com/elves/elvish/blob/master/LICENSE),
> BSD-2-Clause).

Source: <https://github.com/elves/elvish>

## TL;DR

`elvish` is what you reach for when the next pipeline you're about
to write in `bash` would have been three lines of `awk` to parse a
JSON array, two `read` loops to keep state across iterations, and
a `set -e` that you don't trust. Instead of stringly-typed pipes,
`elvish` pipes carry real values: `put [1 2 3] | each {|x| * $x 2}`
emits `2 4 6` as three values, not one whitespace-blob you have to
re-parse downstream. The shell ships with a built-in JSON codec
(`from-json` / `to-json`), exception handling (`try { … } catch e
{ … }`), modules (`use github.com/zzamboni/elvish-modules/alias`),
and a Lispy/Tcl-flavoured syntax that is small enough to learn in
an afternoon. Interactive mode keeps the affordances you expect
from a 2020s shell: directory history (`Ctrl-L`), command history
(`Ctrl-R`) with arrow-key narrowing, and a per-shell location
tracker — all built in, no plugin manager required.

## Install

```bash
# Homebrew (macOS / Linux)
brew install elvish

# Go
go install src.elv.sh/cmd/elvish@v0.21.0

# Linux package managers
# Arch: pacman -S elvish
# Debian / Ubuntu: apt install elvish
# Fedora: dnf install elvish
# Nix: nix-env -iA nixpkgs.elvish
# Alpine: apk add elvish

# Pre-built binaries
curl -Lo elvish.tar.gz "https://dl.elv.sh/darwin-arm64/elvish-v0.21.0.tar.gz"
tar xf elvish.tar.gz
sudo install elvish-v0.21.0 /usr/local/bin/elvish

# verify
elvish -version    # 0.21.0

# Make it your login shell (optional)
which elvish | sudo tee -a /etc/shells
chsh -s "$(which elvish)"

# Or just launch it from inside bash/zsh to try
elvish
```

Config lives in `~/.config/elvish/rc.elv`; modules in
`~/.config/elvish/lib/`. There is no `.elvishrc` in `$HOME`.

## License

BSD-2-Clause — see
[LICENSE](https://github.com/elves/elvish/blob/master/LICENSE).
Permissive, requires copyright notice retention; no advertising
clause, no copyleft.

## One Concrete Example

```elvish
# 1. JSON pipelines without `jq`
curl -s https://api.github.com/repos/elves/elvish/releases |
  from-json |
  each {|r| put $r[tag_name] $r[published_at] } |
  take 5

# 2. typed list comprehension
put [1 2 3 4 5] | each {|x| if (> $x 2) { put (* $x $x) } }
# → 9 16 25

# 3. structured exception handling
try {
  curl -fsSL https://example.com/maybe-down | from-json
} catch e {
  echo "fetch failed:" $e
  exit 1
}

# 4. maps as first-class values
var config = [&host=db.local &port=5432 &ssl=$true]
echo $config[host]:$config[port]

# 5. closures captured by name (no `eval` needed)
fn retry {|n cmd|
  for i (range $n) {
    try { $cmd; return } catch e { sleep (* $i 0.5) }
  }
  fail "gave up after "$n" tries"
}
retry 3 { curl -fsSL https://flaky.example.com }

# 6. interactive history with arrow-key narrowing
# (Ctrl-R then type substring — narrows by frecency, no fzf needed)

# 7. directory history
# Ctrl-L then arrows — in-shell, no zoxide hook required

# 8. talk to bash from inside elvish (gradual migration)
e:bash -c 'source ~/.bashrc && some_legacy_function'
```

## Niche It Fills

**A shell language designed in 2014, not 1989.** `bash` and `zsh`
inherit a stringly-typed, line-oriented model from `sh` (1977) where
the only data type is "bytes separated by `IFS`". `elvish` is the
realisation that if you start over with modern lessons (typed
values, lexical scope, closures, exceptions, modules, a real
parser), you can keep 95 % of POSIX-shell ergonomics for the
interactive case while making the scripting case look like Python
or Tcl rather than a whitespace minefield. The catch is the
ecosystem: every script you copy from Stack Overflow will need
translation, and your `~/.zshrc` plugins won't work.

## Why use it

Three things `elvish` does that POSIX shells do not, that pay back
the switching cost:

1. **Typed pipelines.** A pipeline carries discrete values, not
   bytes — so `from-json | each {|x| put $x[name] }` works without
   `jq` or `awk`, and there is no `IFS` / word-splitting / glob
   class of bug. Numbers, lists, maps, closures, and channels are
   all first-class; the only string-y thing left is the I/O at
   each end of an external process.
2. **Exceptions, not exit codes.** `try { … } catch e { … } finally
   { … }` replaces the `set -e` / `set -o pipefail` / `trap ERR`
   minefield. Any non-zero exit from a builtin or external command
   raises a structured exception you can pattern-match on; `set
   -e`-style "fail on first error" is the default, not an opt-in
   booby trap.
3. **One static binary, no plugin manager.** History search,
   directory jump, autosuggest, syntax highlighting, completion
   are all in the core binary. There is no `oh-my-elvish`
   equivalent because there does not need to be one. Updating is
   `brew upgrade elvish` (or `go install src.elv.sh/cmd/elvish@…`),
   not a 30-plugin rebuild.

For an LLM-CLI workflow where an agent generates a shell script,
`elvish`'s structured-exception model means the agent can reason
about failure modes (`try { … } catch …`) rather than emitting
defensive `if [ $? -ne 0 ]` after every command, and the JSON
codec means a generated tool-call pipeline does not need to bolt
`jq` onto every step.

## Vs Already Cataloged

- **Vs [`fish-shell`](../fish-shell/):** orthogonal philosophies.
  `fish` keeps the POSIX-style stringly-typed pipeline and invests
  in *interactive* ergonomics (autosuggest from history, syntax
  highlighting, friendly error messages, abbrs). `elvish` keeps
  fewer interactive flourishes by default and invests in *scripting*
  ergonomics (typed values, exceptions, modules, closures). Pick
  `fish` if your goal is "a nicer interactive shell"; pick
  `elvish` if your goal is "a shell I can write 200-line programs
  in without reaching for Python".
- **Vs [`nushell`](../nushell/):** the closest peer — both pipe
  typed values, both reject the byte-stream POSIX model. `nushell`
  goes further with table-as-primary-type and a SQL-flavoured
  query DSL (`ls | where size > 1mb | sort-by modified`). `elvish`
  stays closer to a Lispy general-purpose language with lists and
  maps as the core types and no built-in tabular DSL. Choose
  `nushell` if your workload is mostly "tabular data wrangling at
  the shell"; choose `elvish` if your workload is mostly "small
  programs that happen to spawn processes".
- **Vs [`oils`](../oils/):** `oils` (osh / ysh) is the
  "POSIX-compatible upgrade path" — `osh` runs your existing bash
  scripts unchanged, `ysh` is the new typed language. `elvish`
  makes no compatibility promise and is the cleaner-sheet design.
  Pick `oils` if you have a 10k-line bash codebase to migrate
  gradually; pick `elvish` if you are choosing a shell for new
  scripts.
- **Vs `bash` / `zsh` (POSIX):** different problem space. POSIX
  shells win on universal availability (every box has `/bin/sh`),
  on the corpus of existing scripts, and on "the shell my
  coworkers know". `elvish` wins on everything inside the script
  itself. Most users keep `zsh` as login shell on shared servers
  and reach for `elvish` for personal scripts and laptops.

## Caveats

- **Not POSIX-compatible.** Your `.bashrc`, your `oh-my-zsh`
  plugins, your `nvm` / `pyenv` / `rbenv` hooks all assume bash
  syntax and won't work unmodified. Most have `elvish` ports
  (`use github.com/zzamboni/elvish-modules/…`) but the migration is
  per-tool, not automatic.
- **Tab completion is per-binary opt-in.** The core ships
  completion for ~40 common commands; everything else falls back to
  filename completion. The `zzamboni/elvish-completions` module
  closes the gap for ~20 more, but you will hit
  "no completion for this tool" more often than in `zsh` with
  `compinit`.
- **Pre-1.0, syntax can break.** v0.x releases occasionally
  rename builtins or adjust syntax (e.g. v0.16 → v0.17 changed
  the closure capture rules). Pin a version in shared config and
  re-read the migration notes on upgrade.
- **No real job control yet.** `Ctrl-Z` / `bg` / `fg` semantics
  are minimal compared to `bash`; long-running interactive jobs
  are best run under `tmux` or as `nohup` background processes.
- **Smaller community than `fish` / `nushell`.** Stack Overflow
  answers are sparse, third-party modules are mostly one author
  (`zzamboni`), and the chance an LLM generates an
  *almost-correct* `elvish` snippet is lower than for `bash`.
  Verify generated code against the official docs at
  <https://elv.sh/learn/>.
- **Not a drop-in for CI scripts.** GitHub Actions, GitLab CI,
  CircleCI all default to `bash`; switching `shell:` to `elvish`
  works but every reusable action you import will assume bash.
  Use `elvish` for *your* scripts called *from* a bash shim, not
  as the CI shell itself.
