# oils

## What it does
A **POSIX-compatible shell with an upgrade path** — ships two binaries
from one codebase: `osh` (the *Oil shell*) is a near-100% bash-compatible
runtime that runs your existing `.bashrc`, build scripts, autotools
output, and `set -euo pipefail` pipelines unchanged but with cleaner
error messages and stricter parsing; `ysh` (the *YSH language*, formerly
Oil) is a from-scratch shell language with a real type system (`Int`,
`Str`, `List`, `Dict`, `Bool`, `Null`), JSON / J8-Notation as
first-class values, expression-vs-command modes, structured arrays
without `IFS` footguns, exceptions instead of `$?` checking, garbage
collection, and a parser that produces a real AST you can dump with
`osh -n`. The same process boots either dialect via shebang
(`#!/usr/bin/env osh` vs `#!/usr/bin/env ysh`); inside `osh` you can
opt into YSH features piecemeal with `shopt --set ysh:upgrade` so a
codebase can migrate file-by-file. Distributed as a single tarball that
configures + makes a static binary on any Unix with a C compiler — no
Python at runtime despite the long Python-prototype history.

## Why it's interesting
Different shape from bash / zsh / dash (entrenched POSIX shells with
decades of accumulated quoting, IFS, and word-splitting footguns;
oils-osh is bash-bug-compatible but adds a real parser and better
diagnostics, oils-ysh replaces the language entirely), from
[`fish-shell`](../fish-shell/) (interactive-first, deliberately not
POSIX-compatible, scripts don't port either way), from
[`nushell`](../nushell/) (structured-data-first shell built around typed
tables and a dataframe-like pipeline; great for data work but not a drop-
in for `bash` build scripts), from [`amber-shell`]
(transpile-to-bash language — produces shell scripts as build output;
oils-ysh is a runtime, not a transpiler), and from `xonsh` (Python
sprinkled with shell — runtime is CPython, perf and dependency cost
match). oils is the *POSIX-bash you can keep + a clean shell language
you can grow into, in one binary* shape: pick it specifically when you
own bash codebases that you want to keep running today but stop writing
new bash for tomorrow, or when you want a shell whose error messages
actually point at the offending token. Do **not** pick it as your
interactive login shell on day one (interactive UX still trails
fish/zsh+plugins), for Windows-native scripts (Unix-only), or as a
sandboxed embedded scripting language (use Lua, Starlark, or
[`pkl`](../pkl/)).

## Niche category
Bash-compatible POSIX shell with an opt-in upgrade language —
`osh` runs existing bash, `ysh` is a typed JSON-native shell.

## Repo
https://github.com/oils-for-unix/oils
(canonical mirror; primary development happens on the upstream's own
infrastructure with GitHub used for issue tracking and CI mirroring)

## Version pinned
`0.24.0` (latest tarball release at https://www.oilshell.org/release/0.24.0/ ,
published 2024-11-10)

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE.txt`

## Install
```sh
# Source tarball (works on any Unix with a C compiler — recommended path)
wget https://www.oilshell.org/download/oils-for-unix-0.24.0.tar.gz
tar -xzf oils-for-unix-0.24.0.tar.gz
cd oils-for-unix-0.24.0
./configure
make
sudo ./install   # installs osh + ysh into /usr/local/bin

# Homebrew (macOS / Linux)
brew install oils-for-unix

# Nix
nix profile install nixpkgs#oils-for-unix

# Arch (AUR)
yay -S oils-for-unix
```

## Usage examples
```sh
# Run an existing bash script unchanged under osh (catches more errors at parse time)
osh ./build.sh

# Lint a bash script without executing it (pretty-print the AST)
osh -n ./build.sh

# Drop into an interactive osh shell
osh

# YSH: typed values + JSON-native + expression mode
ysh -c '
  var users = json read (./users.json)
  for u in (users) {
    if (u.active) {
      echo $[u.name]
    }
  }
'

# Migrate a bash script file-by-file: keep the shebang as bash, opt into YSH features
cat > deploy.sh <<'YSH'
#!/usr/bin/env osh
shopt --set ysh:upgrade   # turn on strict parsing, typed vars, error-on-unset
var hosts = :| web-1 web-2 web-3 |
for h in (hosts) {
  ssh $h 'systemctl restart app'
}
YSH
osh deploy.sh
```
