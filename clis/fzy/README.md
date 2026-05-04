# fzy

> **The minimalist fuzzy finder** — a small C TTY filter (~1500
> lines of code) that reads lines on stdin, draws a tiny in-place
> picker over the bottom of your terminal, ranks candidates with
> a transparent published scoring algorithm, and writes the
> chosen line to stdout. Pinned to **v1.0**
> ([LICENSE](https://github.com/jhawthorn/fzy/blob/master/LICENSE),
> MIT).

Source: <https://github.com/jhawthorn/fzy>

## TL;DR

`fzy` is `fzf`'s smaller, older, more opinionated sibling. The
contract is identical — pipe lines in, get the chosen one out:
`find . -type f | fzy` prints the path you picked. Where `fzy`
differs is in scope and aesthetics. The whole tool is ~1500 lines
of C with no external dependencies (curses is optional and built
in), the binary is well under 100 KB, the picker draws **only**
the lines it needs (default 10) at the bottom of the terminal
without taking over the screen, and the matching algorithm is a
documented Smith-Waterman-style score that explicitly favours
matches at word boundaries, after path separators, and in
camelCase humps — published in `ALGORITHM.md` so it can be
reasoned about, not just felt. There are no preview panes, no
multi-select hotkeys, no shell key-binding scripts, no plugin
system. One job, well done, with a deliberately small surface.

## Install

```bash
# Homebrew
brew install fzy

# Debian / Ubuntu
sudo apt install fzy

# Fedora
sudo dnf install fzy

# Arch
sudo pacman -S fzy

# Alpine
sudo apk add fzy

# from source (any Unix with a C compiler)
git clone https://github.com/jhawthorn/fzy.git
cd fzy && make && sudo make install

# verify
fzy --version    # 1.0 (c) 2014-2018 John Hawthorn
```

No shell-init step is required. To wire `fzy` into a shell
binding (Ctrl-T → file picker, Ctrl-R → history search, Alt-C →
cd picker, Tab → completion), the upstream repo ships
`contrib/fzy-zsh`, `contrib/fzy-bash`, and `contrib/fzy-fish`
under 100 lines each — drop one into your rc and source it.

## License

MIT — see
[LICENSE](https://github.com/jhawthorn/fzy/blob/master/LICENSE).
Permissive, no attribution needed for binaries; redistribute
freely.

## One Concrete Example

```bash
# 1. the canonical pipeline — pick a file, then act on it
vim "$(find . -type f -not -path '*/.*' | fzy)"

# 2. cd to a picked directory (alias-friendly)
cd "$(find . -type d -not -path '*/.*' | fzy)"

# 3. checkout a git branch interactively
git branch --format='%(refname:short)' | fzy | xargs git checkout

# 4. kill a process by picking it
ps -o pid,user,comm,args -ax | fzy | awk '{print $1}' | xargs kill

# 5. fuzzy-search shell history (no plugin needed)
HISTFILE=~/.zsh_history fc -ln 1 | fzy

# 6. pre-fill the query (`-q`) and pre-select a count (`-l`)
ls /usr/local/bin | fzy -q 'git' -l 20

# 7. non-TTY mode — score and rank without drawing UI
#    (good for testing the algorithm in scripts)
echo -e "src/main.go\nsrc/foo/main.go\ntest/main_test.go" | \
    fzy -e main
# ==> prints the best-scoring single match (src/main.go)

# 8. show match scores for debugging the ranking
echo -e "src/main.go\nsrc/foo/main.go\ntest/main_test.go" | \
    fzy -s -q main
# ==> each line prefixed with the numeric score

# 9. inside the picker:
#    type to filter            tab/down  next candidate
#    shift-tab/up  prev        enter     accept
#    ctrl-c / esc  cancel      ctrl-w    delete word
#    ctrl-u  clear query       ctrl-d/h  delete char
```

## Niche It Fills

**A fuzzy picker that does *one* thing and stays small.** `fzf`
has accumulated ~15 years of features (preview windows,
multi-select, mouse, tmux popup, history-mode, completion-mode,
plugin hooks, --ansi, --tabstop, --tmux, --listen) — for many
operators that surface area is the point, but it makes `fzf` the
opposite of small. `fzy` is a deliberate counter-position: the
narrow Unix filter ("read lines, show picker, write line") with
a documented matching algorithm and a footprint that fits in a
busybox container or an embedded Linux build without bringing in
a 4 MB Go binary plus its bash plugin tree. For an LLM-CLI
workflow that wants an *interactive disambiguation step* in the
middle of a shell pipeline (the agent emits 12 candidate paths;
the operator picks one; the path goes into the next tool call),
`fzy`'s minimal contract is exactly the right shape.

## Vs Already Cataloged

- **Vs [`fzf`](../fzf/):** the dominant alternative and the
  one most operators install first. `fzf` ships preview windows,
  multi-select with `--multi`, ANSI colour passthrough, a tmux
  popup mode, a server mode (`--listen`), shell-completion
  integrations for ~10 shells, and a plugin culture (vim,
  neovim, helix) that has standardised around it. `fzy` ships
  none of those — the trade is "every feature you might want" vs
  "a small, audited C binary with one job". Pick `fzf` when
  preview / multi-select / tmux / completion are part of the
  workflow; pick `fzy` when the appeal is exactly the absence of
  those moving parts (constrained environments, security audits,
  reproducible-build pipelines, busybox containers, or just
  taste).
- **Vs [`skim`](../skim/):** `skim` is the Rust answer in the
  same niche as `fzf` — preview, multi-select, ANSI, shell hooks.
  `fzy` is smaller still and predates both. Pick `skim` for the
  Rust ecosystem with `fzf`-class features; pick `fzy` for a
  handful of KB of C.
- **Vs [`peco`](../peco/):** `peco` is the older Go interactive
  filter, similar size class to `fzy` but Go-runtime-shaped (a
  ~3 MB binary instead of a sub-100-KB one). Comparable
  philosophy ("pick a line, write it"), different language tax;
  on a busybox / Alpine / embedded box `fzy` is the right pick,
  on a Go-heavy desktop `peco` is fine.
- **Vs [`gum`](../gum/):** `gum` is the *prompt-toolkit-as-CLI*
  for shell-script UX (spinners, inputs, choosers, confirmations)
  — `gum choose` is the closest verb to `fzy`, but `gum`'s scope
  is the whole shell-script-UI surface, where `fzy` is just the
  picker. Pick `gum` when the shell script also wants spinners
  and styled prompts; pick `fzy` when picker is all you need.
- **Vs [`zoxide`](../zoxide/) / [`atuin`](../atuin/) /
  [`mcfly`](../mcfly/):** orthogonal — those are *opinionated
  consumers* of fuzzy picking (directory frecency, shell history,
  history neural ranking) that often *embed* `fzf`/`skim` for
  their picker. `fzy` is the *picker primitive* underneath. They
  compose: `zoxide query --list | fzy` is a one-liner equivalent
  of `zi`.

## Caveats

- **Last upstream tag is v1.0 (2018).** The project is
  intentionally feature-complete and the maintainer is
  conservative about adding scope; "stale" by GitHub-stars
  metrics, but the ~1500 lines of C have outlived most of its
  contemporaries unchanged. If you need a tool that ships
  monthly releases, `fzf` and `skim` are the active choices.
- **No preview pane.** `fzy` deliberately does not implement
  `fzf --preview`. The conventional workaround is to do the
  preview *after* the pick: `f=$(... | fzy) && bat "$f"`. If
  preview-while-filtering is part of the workflow, this is a
  hard miss and you want `fzf`.
- **No multi-select.** Single-line output only. If the workflow
  is "pick 5 of these 50", `fzf --multi` (or `gum choose
  --no-limit`) is the right tool; `fzy` is single-pick by
  design.
- **No native shell-history widget.** Unlike `fzf`'s
  `Ctrl-R` rebinding, `fzy` ships only the contrib snippets
  (`contrib/fzy-zsh` etc.) for shell hooks. They work fine, but
  the ergonomics of "set this env var and run `fzf --bash`" is
  not present — wiring is the operator's job.
- **No `--ansi` colour passthrough.** Lines with ANSI escapes
  in them render as raw escape codes, not coloured text. `fzy`'s
  picker itself uses minimal colour by design; if your pipeline
  produces coloured input you want to preserve, strip with
  `sed 's/\x1b\[[0-9;]*m//g'` before piping in, or use `fzf`.
- **TTY-bound.** Like every interactive picker, `fzy` requires a
  controlling TTY; in a non-interactive context (CI, `nohup`,
  `&` background) it errors out. The `-e <query>` non-TTY mode
  exists exactly for the "score and pick best match
  programmatically" case where the script knows the query in
  advance and just wants `fzy`'s ranker.
