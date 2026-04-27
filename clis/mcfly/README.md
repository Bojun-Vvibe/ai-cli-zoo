# mcfly

> **Fly through your shell history. Great Scott!** — a single
> Rust binary that replaces `Ctrl-R` (the default reverse-i-search
> in bash/zsh/fish) with a small neural-network-ranked, context-aware
> suggestion list backed by a SQLite history database. Pinned to
> **v0.9.4** (commit
> `17f03db47b5ba9141d7b5df9f652a421b6fb99be`,
> [LICENSE](https://github.com/cantino/mcfly/blob/master/LICENSE),
> MIT).

Source: <https://github.com/cantino/mcfly>

## TL;DR

`mcfly` hooks your shell, intercepts `Ctrl-R`, and replaces the
linear chronological back-search with a ranked TUI: matches are
scored by a small bundled neural network on lexical similarity to
the typed query, the directory you're currently in, the exit
status of the past command, the time of day, and how recently the
command ran. The result: the command you actually want is usually
the first hit, not the seventeenth, and the picker filters as you
type instead of jumping to a single match.

## Install

```bash
# Homebrew (macOS / Linux)
brew install mcfly

# Cargo (any OS with a Rust toolchain)
cargo install mcfly

# Linux package managers
# Arch:    pacman -S mcfly
# Nix:     nix-env -iA nixpkgs.mcfly

# install the shell hook (one-time, per shell)
# bash:
echo 'eval "$(mcfly init bash)"' >> ~/.bashrc
# zsh:
echo 'eval "$(mcfly init zsh)"' >> ~/.zshrc
# fish:
echo 'mcfly init fish | source' >> ~/.config/fish/config.fish

exec $SHELL

# verify
mcfly --version    # mcfly 0.9.4
```

The init eval is what rewires `Ctrl-R`; without it, the binary
sits unused. First launch backfills the existing shell history
into `~/.local/share/mcfly/history.db`.

## License

MIT — see
[LICENSE](https://github.com/cantino/mcfly/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. press Ctrl-R from any prompt — TUI opens with ranked history
#    (start typing to filter; Up/Down to pick; Enter to run; Tab to
#     edit before running)

# 2. directory-aware ranking: cd into a project, Ctrl-R, type "test"
#    → commands you ran in *this* directory float to the top
cd ~/myrepo
# Ctrl-R, type: test
# top hit: `cargo test --workspace` (last run here, exit 0)
# below:   `pytest -k auth`           (last run in another repo)

# 3. exit-status weighting: commands that exited non-zero are
#    de-ranked, so a typo doesn't keep haunting you
ls /no/such/path           # exits 2
# next Ctrl-R, type "ls /no" → typo is buried under successful matches

# 4. browse mode: see history without searching
mcfly search ""            # opens TUI on full history

# 5. import existing history before installing the hook
mcfly init zsh             # (the eval) auto-imports first time

# 6. delete a sensitive command from history (TUI: F2 on the entry)
#    or directly:
sqlite3 ~/.local/share/mcfly/history.db \
  "delete from commands where cmd like '%AWS_SECRET%';"

# 7. tune ranking weights
export MCFLY_FUZZY=2          # allow up to 2 typos in matches
export MCFLY_RESULTS=50       # show 50 results instead of 10
export MCFLY_INTERFACE_VIEW=BOTTOM   # picker at bottom of screen
export MCFLY_KEY_SCHEME=vim   # j/k navigation
```

## Niche It Fills

**Default `Ctrl-R` is chronological + literal substring; [`fzf`](https://github.com/junegunn/fzf)'s
`Ctrl-R` integration is fuzzy + chronological; [`atuin`](../atuin/)
is a sync-first history database with rich filters.** `mcfly` sits
in a fourth corner: **a neural ranker** that uses *signal* — the
working directory, the exit status, the time, the recency — to
guess which past command best fits the current context. It
doesn't try to sync across machines (atuin's job) and it doesn't
fuzz-match plain text without ranking (fzf's job); it tries to
make the *first* result correct.

For agent / LLM workflows where a model is reading a developer's
shell history to mimic their patterns, `mcfly`'s SQLite database
(`~/.local/share/mcfly/history.db`) is a clean, queryable, exit-coded
log — far richer than the flat `~/.zsh_history` file.

## Vs Already Cataloged

- **Vs [`atuin`](../atuin/):** atuin is the sync-and-query
  history database (E2E-encrypted sync server, full-text + regex
  filters, stats dashboard); mcfly is the *local ranker* —
  smaller scope, no server, no account, but its directory- and
  exit-status-aware scoring usually puts the right command first
  on a single machine. Use atuin when sharing history across
  laptops matters; use mcfly when you live on one box and want
  the picker to be smarter, not bigger.
- **Vs [`fzf`](https://github.com/junegunn/fzf) `Ctrl-R`:** fzf
  fuzzy-matches your literal query against the history file in
  recency order, ignoring whether the command worked or where
  you were. mcfly weights the same matches by working directory,
  exit code, and a learned model. `fzf` wins when you remember
  exactly what you typed; `mcfly` wins when you remember the
  *intent* but not the exact incantation.
- **Vs [`zoxide`](../zoxide/):** complementary — zoxide ranks
  *directories* by frecency for `cd`; mcfly ranks *commands* by
  context-aware frecency for `Ctrl-R`. Pair them: zoxide gets
  you to the right repo, mcfly recalls the right command for
  that repo.
- **Vs the built-in shell `Ctrl-R`:** the default is a literal,
  chronological scan that requires you to remember the start of
  the command. mcfly turns that into a filtered, ranked picker
  with the metadata to surface "the test command for *this*
  project" instead of "the most recent command containing
  'test'".

## Caveats

- **First launch is slow on giant histories.** The initial import
  of `~/.zsh_history` / `~/.bash_history` parses every line and
  builds the SQLite index; on multi-megabyte history files
  expect a 30–60 s pause the first time the shell opens after
  install. Subsequent shells are instant.
- **Ranking quality grows with usage.** The first few weeks the
  picker behaves much like fzf, because there isn't yet enough
  per-directory + exit-status signal to differentiate similar
  commands. The "ranks the right one first" payoff lands after
  a few hundred recorded commands.
- **The shell hook intercepts `Ctrl-R` globally.** If you have
  another binding on `Ctrl-R` (a custom widget, a
  fzf-tab plugin, a tmux key-table conflict), you have to
  unbind that first; mcfly will not chain with another widget on
  the same key.
- **No cross-machine sync.** History stays in
  `~/.local/share/mcfly/history.db` on one host. If you want the
  same history on three machines, that is atuin's problem
  domain, not mcfly's; the two can coexist, but there is no
  built-in mcfly sync.
- **The model is small and bundled.** It is not a transformer or
  an LLM — it is a tiny neural network trained on the public
  history corpus the maintainer collected. It cannot understand
  semantics ("the command that deploys staging") only surface
  features (lexical match + directory + exit + recency). Treat
  the ranking as a smarter heuristic, not a chatbot.
