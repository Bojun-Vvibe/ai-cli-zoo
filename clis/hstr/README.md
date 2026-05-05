# hstr

> **A bash / zsh shell-history TUI bound to Ctrl-R that turns
> the linear `~/.bash_history` into a ranked, filterable,
> deduplicated, colorized suggest-box** — type a few characters,
> arrow through ranked matches, Enter to run, F2 to paste into
> the prompt without running, Del to forget a command (and
> scrub it from history on disk). Pinned to **v3.2** (SPDX:
> `Apache-2.0`,
> [LICENSE](https://github.com/dvorka/hstr/blob/master/LICENSE)).

Source: <https://github.com/dvorka/hstr>

## TL;DR

Default `Ctrl-R` (bash reverse-i-search) is a one-line
substring match with no ranking, no preview of nearby
candidates, and no way to delete an entry. `hstr` replaces it
with a full-screen TUI that:

1. **Ranks** by frequency × recency (the "I-actually-use-this"
   metric) instead of last-seen-only.
2. **Deduplicates** — if you ran `git push origin main` 200
   times, it's one entry, not 200.
3. **Filters live** as you type, with regex / keyword / exact
   substring modes (Ctrl-E to cycle).
4. **Forgets** — Del removes the highlighted command from the
   suggest box *and* scrubs it from `~/.bash_history` on disk
   (the only way to delete a typo'd `--password=hunter2` after
   the fact short of editing the history file by hand).
5. **Colorizes** by command (red for `rm`, green for `git`,
   etc.) so the eye lands on the right line in a 30-row list.

It hooks in as a `Ctrl-R` binding (or `Ctrl-Alt-R`, or any
key) and is otherwise invisible — exits to a normal shell
prompt, no daemon, no history-format conversion.

## Install

```bash
# Homebrew (macOS / Linux)
brew install hstr

# Debian / Ubuntu (PPA)
sudo add-apt-repository ppa:ultradvorka/ppa
sudo apt update && sudo apt install hstr

# Fedora / RHEL
sudo dnf install hstr

# Arch (AUR)
yay -S hstr

# from source
git clone --branch 3.2.0 https://github.com/dvorka/hstr
cd hstr/build/tarball && ./tarball-automake.sh
cd ../.. && ./configure && make && sudo make install

# verify
hstr --version    # hstr 3.2.0
```

Then bind to `Ctrl-R` for your shell:

```bash
# bash — append to ~/.bashrc
hstr --show-bash-configuration >> ~/.bashrc

# zsh — append to ~/.zshrc
hstr --show-zsh-configuration >> ~/.zshrc
```

Open a new shell, hit `Ctrl-R`, type a few letters of any past
command — the suggest box drops in, ranked by your actual use.

## License

Apache-2.0 — see
[LICENSE](https://github.com/dvorka/hstr/blob/master/LICENSE).
Permissive with patent grant; safe for redistribution and
binary packaging.

## One Concrete Example

```bash
# 1. ctrl-R opens the TUI; type to filter live
#    arrow keys move; Enter runs; F2 pastes into prompt

# 2. ranking modes — Ctrl-/ cycles between
#    metric (frequency × recency), recency, history-order

# 3. matching modes — Ctrl-E cycles between
#    exact substring, keywords, regex

# 4. case sensitivity — Ctrl-T toggles

# 5. delete a leaked-password line from history forever
#    arrow to the offending line, Del, confirm — gone from
#    ~/.bash_history on disk

# 6. favorites — Ctrl-F adds the current line to a favorites
#    list that floats to the top of suggestions

# 7. one-shot mode (no shell binding) — query history from a
#    script
hstr --non-interactive 'docker ps' | head -1
```

## Niche It Fills

**Ranked, scrubbable shell-history search.** Default `Ctrl-R`
is a substring match against a flat dedupe-free file; `fzf`'s
history widget (the modern default) sorts by recency only and
cannot delete entries. `hstr` is the only widely-packaged tool
that does **frequency × recency ranking**, **on-disk delete**,
and **per-command colorization** in one TUI bound to the
canonical `Ctrl-R` keystroke.

## Why use it

1. **Frequency × recency ranking.** The command you ran 50
   times last week ranks above the command you ran once this
   morning — the opposite of recency-only sort, and the right
   default for muscle memory.
2. **Deduplication.** 200 invocations of `git push origin main`
   collapse to one row; the suggest box stays scannable.
3. **On-disk scrub.** Del removes a line from
   `~/.bash_history` on the spot. The standard alternative is
   `history -d $LINE && history -w` plus manual `~/.bash_history`
   editing — no tool other than `hstr` makes this a single
   keystroke.
4. **Colorized by command.** `rm`, `git`, `docker`, `kubectl`
   each get a distinct hue. In a 30-row suggest box this is
   the difference between "scan in 1 second" and "read every
   line."
5. **Drops onto every distro.** Packaged in Homebrew, apt
   (PPA), dnf, AUR; single C binary, no runtime, ~200 KB.

## Vs Already Cataloged

- **Vs [`atuin`](../atuin/):** atuin is the modern,
  ambitious replacement — SQLite-backed history, end-to-end
  encrypted sync across machines, exit-status / cwd / duration
  capture, full-text search. It *replaces* `~/.bash_history`
  with its own store. `hstr` keeps the plain
  `~/.bash_history` file as the source of truth and only
  improves the search-and-prune UX on top of it. Use atuin
  if you want sync + rich metadata; use `hstr` if you want a
  drop-in `Ctrl-R` upgrade with zero new state and zero
  network surface.
- **Vs [`mcfly`](../mcfly/):** mcfly also re-ranks history
  (neural-net relevance) but does not delete entries from
  disk and does not colorize by command. `hstr`'s ranking is
  simpler (deterministic frequency × recency, no model) and
  its kill-feature (Del to scrub) has no equivalent in mcfly.
- **Vs `fzf` history widget:** fzf is recency-sort only and
  read-only against history. `hstr` ranks by use *and* writes
  back. Many users keep both — fzf for fuzzy file finding,
  hstr for `Ctrl-R`.

## Caveats

- **bash and zsh only.** No fish (fish has its own ranked
  history), no nushell, no PowerShell.
- **History file must be plain `~/.bash_history` /
  `~/.zsh_history`.** Tools that replace the on-disk format
  (atuin, fish) take that source away from `hstr`.
- **Del is destructive and immediate.** A scrubbed line is
  gone from disk; back up `~/.bash_history` before bulk-delete
  sessions.
- **No cross-machine sync.** History stays on the local file.
  Pair with `chezmoi` / `tuckr` / `rsync` if you want to ship
  history between hosts, or use `atuin` for that purpose.
