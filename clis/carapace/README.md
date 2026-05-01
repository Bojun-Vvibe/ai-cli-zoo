# carapace

> **One completion engine, every shell** — a single Go binary
> that ships hand-written argument completers for ~1100 commands
> (git, kubectl, docker, gh, terraform, helm, npm, cargo, rg, fd,
> jq, …) and emits them as native completion scripts for bash,
> zsh, fish, nushell, elvish, oil, xonsh, tcsh, and PowerShell —
> so the *same* completion definition gives you flag descriptions,
> positional value lists, and subcommand trees in whichever shell
> you happen to be in. Pinned to **v1.6.5** (commit
> `5ec341d81f3289ff008c4e3e601e392f188d79b9`,
> [LICENSE](https://github.com/carapace-sh/carapace-bin/blob/v1.6.5/LICENSE),
> MIT).

Source: <https://github.com/carapace-sh/carapace-bin>

## TL;DR

`carapace` solves the "every shell reinvents tab-completion" problem.
Instead of hunting for `_kubectl` zsh completion in one repo, a
fish completion in another, and giving up on nushell entirely, you
install one binary, drop one line into your shell rc, and every
supported command (the [spec list](https://carapace-sh.github.io/carapace-bin/spec.html)
covers ~1100 of them) gets rich completion: flag names *with
descriptions on the right*, value enums (`kubectl get <TAB>` lists
your real namespaces by shelling out to `kubectl`), positional
arguments, mutually-exclusive flag groups, and subcommand trees.
The completers are declared in a unified spec format, so adding a
new tool or extending an existing one is one YAML file, not three
shell-specific scripts.

## Install

```bash
# Homebrew (macOS / Linux)
brew install carapace

# Scoop (Windows)
scoop install carapace

# Nix
nix-env -iA nixpkgs.carapace

# Arch
pacman -S carapace-bin

# Pre-built binary (any OS)
curl -Lo carapace.tar.gz "https://github.com/carapace-sh/carapace-bin/releases/download/v1.6.5/carapace-bin_1.6.5_darwin_arm64.tar.gz"
tar xf carapace.tar.gz
sudo install carapace /usr/local/bin/

# Go install
go install github.com/carapace-sh/carapace-bin/cmd/carapace@v1.6.5

# verify
carapace --version    # 1.6.5

# Wire into your shell — pick one:
# ~/.bashrc
source <(carapace _carapace bash)
# ~/.zshrc  (must come AFTER `compinit`)
source <(carapace _carapace zsh)
# ~/.config/fish/config.fish
carapace _carapace fish | source
# ~/.config/nushell/config.nu  — see `carapace _carapace nushell`
# ~/.config/elvish/rc.elv
eval (carapace _carapace elvish | slurp)
# PowerShell
carapace _carapace powershell | Out-String | Invoke-Expression
```

## License

MIT — see
[LICENSE](https://github.com/carapace-sh/carapace-bin/blob/v1.6.5/LICENSE).
Permissive, no attribution required for binaries; redistribute
freely.

## One Concrete Example

```bash
# 1. flag completion with descriptions on the right (zsh / fish / nushell)
$ kubectl get pods --<TAB>
--all-namespaces        -- list across all namespaces
--field-selector=       -- selector to filter on, supports '=', '==', '!='
--label-selector=       -- selector (label query) to filter on
--output=               -- output format (json|yaml|wide|name|...)
--show-labels           -- show all labels in the last column
...

# 2. positional value enums sourced from the live system
$ kubectl get pods -n <TAB>
default                 kube-system             ingress-nginx
my-app-staging          my-app-prod             monitoring

# 3. subcommand tree introspection
$ git <TAB>
add        bisect     branch     checkout   clone      commit
diff       fetch      log        merge      pull       push
rebase     reset      revert     show       stash      status
tag        worktree   ...

# 4. dynamic completion that shells out
$ docker exec <TAB>          # lists running container names + IDs
$ ssh <TAB>                  # lists ~/.ssh/config Host entries + known_hosts
$ gh pr checkout <TAB>       # lists open PRs in the current repo

# 5. spec-driven completion for your own tools
$ cat ~/.config/carapace/specs/mytool.yaml
name: mytool
description: my internal CLI
flags:
  --env=: target environment
positional:
  - name: command
    completion:
      command: ["mytool", "list-commands"]
persistentflags:
  --verbose: verbose output
$ mytool --env=<TAB>         # works in every supported shell instantly

# 6. lazy-loaded "bridge" mode — fall back to the shell's own
#    completion when carapace doesn't have a spec for the command
$ export CARAPACE_BRIDGES='zsh,bash,inshellisense,fish,zsh'
# now `mytool <TAB>` will try carapace, then zsh-builtin, then bash, ...

# 7. inspect what carapace knows about a command
$ carapace --list | head -10
$ carapace kubectl --style "carapace.Value=blue,carapace.Description=gray" \
  get pods --
```

## Niche It Fills

**Argument completion as a polyglot, command-centric library — not
a shell-specific cottage industry.** Traditional shell completion
is duplicated work: someone writes `_git` for zsh, someone else
writes `__git_complete` for bash, fish has its own `complete -c
git` calls, nushell has nothing, and your in-house CLI has none of
the above. `carapace` flips it: completions are spec'd once per
command and rendered per shell, so adopting fish or nushell as a
daily driver no longer means losing tab-completion for half your
toolbox. For an LLM agent that wants to discover available
subcommands or flag names of an arbitrary CLI on the user's path,
`carapace <cmd> --` lists them in a parseable shape without
running the tool's `--help` and parsing prose.

## Why use it

Three things `carapace` does that per-shell completion scripts do
not, that pay back the install cost:

1. **One spec, every shell.** When you switch shells (or dual-boot
   between zsh on the laptop and nushell in tmux), your completion
   coverage doesn't change. The completer for `kubectl get pods -n
   <TAB>` lists the same live namespaces in fish as it does in
   zsh, because both shells call the *same* binary that shells out
   to `kubectl get namespaces` and feeds the result back through
   the shell's native completion protocol.
2. **Descriptions on the right, not just names.** Most stock
   completion shows `--field-selector` as a bare token; `carapace`
   ships the description ("selector to filter on") and the right
   shell's renderer (zsh's `_describe`, fish's `complete -d`,
   nushell's `extern`) puts it next to the candidate. That alone
   eliminates a `--help | grep` for the long tail of flags you
   only use monthly.
3. **Bridges + spec authoring make the long tail tractable.**
   `CARAPACE_BRIDGES=zsh,fish,bash` lets the binary fall back to
   whichever shell completion *does* exist for an unsupported
   command, so installing `carapace` strictly increases your
   coverage instead of replacing what you already had. For
   internal CLIs, a 20-line YAML spec under
   `~/.config/carapace/specs/` gives you full multi-shell
   completion without writing a `_mytool` zsh function.

## Vs Already Cataloged

- **Vs [`atuin`](../atuin/):** orthogonal — `atuin` replaces shell
  *history* (every command you ran) with synced fuzzy-searched
  storage; `carapace` enriches *forward* completion (what
  arguments could come next on this command line). They cover
  non-overlapping ergonomic wins: `atuin` answers "what did I
  type", `carapace` answers "what could I type next". Most heavy
  CLI users want both.
- **Vs [`zoxide`](../zoxide/):** orthogonal — `zoxide` does
  frecency-ranked directory jumping (`z proj`); `carapace`
  completes argument tokens once you've already typed
  `kubectl get pods -n `. The verbs don't overlap.
- **Vs [`fish-shell`](../fish-shell/):** complementary — fish has
  the best stock completion of any shell, but it's fish-only.
  `carapace` gives bash / zsh / nushell / elvish the same caliber
  of descriptive, dynamic completion, and inside fish itself it
  fills coverage holes for tools fish doesn't ship a `complete -c`
  file for.
- **Vs `bash-completion` / `zsh-completions` (not cataloged):**
  the closest peers — community repos of per-shell completion
  scripts. `carapace` is more uniform (one spec format, all
  shells), more discoverable (`carapace --list` enumerates
  supported commands), and it gets descriptions onto the
  completion menu in shells that support them, which the static
  `_command` zsh files frequently do not.

## Caveats

- **Coverage is wide but not 100%.** ~1100 commands is a lot, but
  not everything. Use `carapace --list | grep mycli` to check;
  for misses, write a 20-line spec or rely on
  `CARAPACE_BRIDGES=...` to fall back to whatever native
  completion already exists.
- **Zsh ordering is fragile.** `source <(carapace _carapace zsh)`
  must come *after* `autoload -U compinit && compinit`, otherwise
  the completion system isn't initialized yet and the `compdef`
  calls silently no-op. The `carapace` docs are explicit about
  this; `.zshrc` rearrangement is a one-time fix.
- **Dynamic completers shell out — and that has cost.** When
  `kubectl get pods -n <TAB>` lists namespaces, `carapace` is
  actually running `kubectl get namespaces` under the hood. On
  slow clusters or unreachable contexts that adds latency to your
  TAB key. Most users tolerate it; if you don't, the
  `--cache-timeout` flag and per-spec disabling of dynamic
  completers exist.
- **Spec authoring is a small DSL to learn.** YAML keys
  (`positional`, `flags`, `completion.command`,
  `completion.directories`, `persistentflags`, mutually exclusive
  groups) are documented but not Bash-literate. For a single
  internal CLI it's usually faster than writing the equivalent
  zsh `_mytool` function; for a one-off shell alias, completion
  via `alias` propagates from the underlying command for free.
- **History of repo moves.** The project lived under
  `rsteube/carapace-bin` until 2024, then moved to
  `carapace-sh/carapace-bin`. Old install commands and references
  in third-party blogs may use the previous URL — both still
  resolve to the same code, but `go install` paths and `brew
  formulae` are now under `carapace-sh`.
- **Not a shell.** `carapace` does not implement a REPL, history,
  or line editing — it only generates *completion* output for
  whichever shell is calling it. You still pick your daily-driver
  shell separately.
