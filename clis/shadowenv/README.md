# shadowenv

> **A directory-scoped environment-variable manager that
> uses a *shadow stack* — when you `cd` into a project,
> a Lisp-dialect (`Ketos`) script declares the env vars
> that should appear, and shadowenv records which keys it
> set so it can perfectly reverse them on `cd ..`** — no
> "leftover `RUBY_VERSION=3.1` after I left that repo"
> drift, no `direnv reload` foot-gun. Pinned to **v3.4.0**
> ([LICENSE](https://github.com/Shopify/shadowenv/blob/main/LICENSE),
> MIT).

Source: <https://github.com/Shopify/shadowenv>

## TL;DR

`direnv` is the standard answer for "load env vars when I
`cd` into a directory", but its model is *additive*: it
runs your `.envrc`, dumps the resulting environment, and
diffs against the parent. Anything your `.envrc` did via
side-effect (`PATH=...:$PATH`, `unset FOO`, sourcing
another script) is reverse-engineered from that diff,
which can leave stragglers when scripts mutate env in
unusual ways.

`shadowenv` flips it: every mutation goes through a typed
API (`env/set`, `env/prepend-to-pathlist`,
`env/remove-from-pathlist`, `provide`) inside a sandboxed
Ketos (Lisp-on-Rust) script. Because the runtime *knows*
which keys it touched and what their previous values were,
unloading is exact — same env back, byte-for-byte. The
result is a deterministic per-project environment that
composes cleanly across nested directories and survives
shells getting forked / suspended / re-entered.

## Install

```bash
# macOS / Homebrew
brew install shadowenv

# Linux x86_64 (release binary)
curl -L https://github.com/Shopify/shadowenv/releases/download/3.4.0/shadowenv-x86_64-unknown-linux-gnu \
    -o /usr/local/bin/shadowenv
chmod +x /usr/local/bin/shadowenv

# macOS arm64 (release binary)
curl -L https://github.com/Shopify/shadowenv/releases/download/3.4.0/shadowenv-aarch64-apple-darwin \
    -o /usr/local/bin/shadowenv
chmod +x /usr/local/bin/shadowenv

# Cargo (build from source)
cargo install --locked --version 3.4.0 shadowenv

# verify
shadowenv --version    # shadowenv 3.4.0
```

Then add the shell hook to your rc file:

```bash
# zsh
eval "$(shadowenv init zsh)"

# bash
eval "$(shadowenv init bash)"

# fish
shadowenv init fish | source
```

The hook installs a `precmd` (zsh) / `PROMPT_COMMAND`
(bash) callback that runs `shadowenv hook` before every
prompt — the actual loading is one syscall plus one file
read when nothing changed, so prompt latency is in the
sub-millisecond range.

## Use it for

```bash
# Initialize a new project
mkdir myproj && cd myproj
shadowenv trust    # whitelist this directory's shadowenv
mkdir .shadowenv.d
cat > .shadowenv.d/500_ruby.lisp <<'EOF'
(provide "ruby" "3.3.5")
(env/set "BUNDLE_PATH" "vendor/bundle")
(env/prepend-to-pathlist "PATH" "vendor/bundle/bin")
EOF

# Now any new shell that cd's into myproj/ sees:
echo $BUNDLE_PATH         # vendor/bundle
which bundle              # /path/to/myproj/vendor/bundle/bin/bundle

cd ..
echo $BUNDLE_PATH         # (unset; original env restored)

# Inspect what shadowenv would do without applying it
shadowenv diff            # show added/removed/changed vars vs parent dir
shadowenv diff -v         # same, with values

# Run a one-off command inside the shadowenv of a directory
shadowenv exec --dir=/path/to/proj -- bundle exec rake test

# Trust a directory after pulling new shadowenv config from upstream
shadowenv trust

# Show what's currently active
shadowenv hook --shellpid=$$ --fieldwidths=...   # what the prompt hook would emit
```

`provide` is the convention layer: declaring `(provide
"ruby" "3.3.5")` doesn't itself install Ruby — it sets
`SHADOWENV_FORMAL_VERSION_PROVIDED_RUBY=3.3.5`, which a
companion tool (chruby, asdf shim, custom script) reads
to actually pick the runtime. The split keeps shadowenv
out of the "which package manager?" debate.

## Why include it in a CLI catalog

1. **It is the most rigorous answer to "per-project
   env vars" on the market.** `direnv` is the famous
   answer; shadowenv is the *correct* answer for codebases
   where you cannot tolerate environment drift between
   sessions. Shopify built it because their Ruby monorepo
   has dozens of components each wanting different
   `BUNDLE_PATH` / `RUBYOPT` / linker flags, and the
   "additive then diff" model leaked when scripts cleared
   variables. The shadow-stack invariant — `cd ../..`
   restores byte-identical environment — is something
   `direnv` cannot guarantee by construction.
2. **Trust model is conservative-by-default.** Cloning a
   repo gives you no implicit code execution; the first
   `cd` into an untrusted directory prints a clear
   message and refuses to load. `shadowenv trust` records
   a per-directory signature; any future change to
   `.shadowenv.d/*.lisp` reverts to untrusted until you
   re-trust. This is the same threat model as `direnv
   allow`, but with explicit per-file hashes.
3. **Lisp-as-config buys real composition.** Because
   the config is a programming language (Ketos, a
   Rust-embedded Lisp), per-machine overrides, conditional
   blocks, and reusable functions work without escaping
   into shell. Files in `.shadowenv.d/` load in
   lexicographic order, so `100_base.lisp` →
   `500_ruby.lisp` → `900_overrides.lisp` is a stable
   layering convention you can rely on.

For an LLM-CLI workflow, shadowenv is the substrate that
keeps `aider` / `claude-code` / `codex` from leaking the
wrong `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` between
client repos: declare them in `.shadowenv.d/`, and the
key only exists inside that directory tree.

## Vs Already Cataloged

- **Vs [`direnv`](../direnv/):** the same niche.
  shadowenv's reversal is exact; direnv's is best-effort.
  shadowenv requires a Lisp dialect to author config;
  direnv uses bash. direnv has the larger ecosystem
  (per-language layouts, integrations) and is more widely
  supported across IDEs / shells. Pick direnv for ergonomics
  + ecosystem; pick shadowenv when reversal correctness
  is load-bearing (production-shaped dev envs, secrets
  scoped to one repo, language-runtime version dispatch
  in a polyglot monorepo).
- **Vs [`mise`](../mise/) / [`pkgx`](../pkgx/) /
  [`devbox`](../devbox/):** orthogonal — those are
  *runtime* managers (they install the actual Ruby /
  Node / Python). shadowenv only flips env vars; it
  *delegates* runtime selection to whatever reads
  `SHADOWENV_FORMAL_VERSION_PROVIDED_*`. The intended
  combo is shadowenv + chruby (or shadowenv + asdf).
  mise/pkgx/devbox each bundle their own env-loading
  layer, so they are alternatives, not complements.
- **Vs [`dotenvx`](../dotenvx/):** orthogonal — dotenvx
  is `.env`-file loading + encryption. shadowenv is
  a *programmable* env layer with reversal semantics.
  Use dotenvx for "ship one encrypted file"; use
  shadowenv for "compute env vars from a script that
  knows about my OS / arch / git branch".
- **Vs [`teller`](../teller/):** orthogonal — teller
  fetches secrets from cloud vaults and exports them
  for one command. shadowenv is the persistent
  per-directory layer that *would consume* what teller
  fetched. They compose: `shadowenv exec -- teller run
  -- ./script` works.

## Caveats

- **Lisp-dialect learning curve.** `Ketos` is a small,
  documented Lisp, but it *is* Lisp. Teams allergic to
  parens will reach for direnv even when they would
  benefit from shadowenv's correctness. The catalog of
  built-in functions is short (env/set, env/unset,
  env/append-to-pathlist, env/prepend-to-pathlist,
  env/remove-from-pathlist, expand-path, provide) which
  helps.
- **Untrusted directories print a warning every prompt
  until trusted.** This is intentional but can feel
  spammy when cloning a lot of repos. `shadowenv trust`
  is one command; running it inside a script for many
  cloned dirs at once is supported.
- **No Windows support.** Shadowenv targets Unix shells
  (bash / zsh / fish). PowerShell / cmd are not on the
  roadmap. WSL2 works.
- **Very small ecosystem vs. direnv.** No per-language
  shortcuts (no `use ruby`, no `layout python`); you
  write the Lisp yourself or copy from Shopify's
  examples. This is the deliberate trade for the
  reversal guarantees — there is no "magic" layer that
  could leak state.
- **Reload-on-config-change requires `shadowenv trust`
  again.** Editing `.shadowenv.d/500_ruby.lisp` voids
  the per-directory hash; the next prompt will refuse
  to load until you re-trust. This is the security
  model working as designed, but in fast iteration on
  config it can feel like friction.
- **Last release v3.4.0 (2025-08).** Project is in
  steady-state mode (Shopify uses it internally for
  their Ruby monorepo); not a fast-moving roadmap, but
  also not abandoned. Pin a specific version when
  scripting CI.
