# nnn

> **n³ — a tiny (~120 KB), zero-config, zero-dependency terminal
> file manager in C that runs from a single static binary, browses
> at native filesystem speed, and extends through ~30 official
> shell-script "plugins" (open-with-fzf, batch-rename, du-graph,
> upload-to-0x0, …) instead of an embedded scripting runtime.**
> Pinned to **v5.2 ("Blue Hawaii")**,
> [LICENSE](https://github.com/jarun/nnn/blob/master/LICENSE),
> BSD-2-Clause.

Source: <https://github.com/jarun/nnn>

## TL;DR

`nnn` is the file manager you reach for when the box is *small* —
a 64 MB-RAM VPS, a router shell, an Alpine container, an SSH
session into a bastion that bans `pip install`. It is a single C
binary linked against `ncurses` and nothing else, ~120 KB on
disk, that opens instantly and stays under 5 MB resident even on
million-file directories. The UI is a single-pane list (toggle
to detail / disk-usage / quick-look), keymap is `vim`-ish but
mnemonic (`d`elete, `r`ename, `c`opy, `m`ove, `o`pen-with), and
it ships with **contexts** — four parallel "tabs" `1234` you flip
between with the number keys, each with its own cwd / selection
/ filter, so you can stage a copy across two trees without losing
your place. Extension is done not through a Lua/Python plugin
runtime but through `~/.config/nnn/plugins/` — small POSIX shell
scripts that receive the current selection on stdin and the cwd
in `$PWD`. The official plugin repo (`getplugs`) ships ~30
turn-key ones: `fzopen` (fzf-pick + xdg-open), `preview-tui`
(side-pane preview via `tmux`), `nuke` (mime-aware open),
`mocq` / `mocplay` (audio queue), `cbcopy` / `cbpaste`
(clipboard ↔ selection), `bulknew`, `gitroot`, `mtpmount`, etc.

## Install

```bash
# Homebrew (macOS / Linux)
brew install nnn

# Debian / Ubuntu
sudo apt-get install nnn

# Arch
sudo pacman -S nnn

# Alpine (the canonical "tiny container" use case)
apk add nnn

# build from source — needs only a C compiler + ncurses headers
git clone --depth 1 -b v5.2 https://github.com/jarun/nnn.git
cd nnn && make O_NERD=1   # O_NERD enables Nerd Font icons

# pull the official plugin pack
sh -c "$(curl -Ls https://raw.githubusercontent.com/jarun/nnn/master/plugins/getplugs)"

# verify
nnn -V    # 5.2
```

## License

BSD-2-Clause — see
[LICENSE](https://github.com/jarun/nnn/blob/master/LICENSE).
Permissive: redistribute, fork, vendor into commercial products
freely; only the copyright notice must travel with the source.

## One Concrete Example

```bash
# 1. drop into a directory and have the parent shell `cd` to it on exit
#    (nnn writes the cwd to a file the wrapper sources)
cat >> ~/.zshrc <<'EOF'
export NNN_FIFO=/tmp/nnn.fifo          # enable plugin pipe
export NNN_PLUG='p:preview-tui;f:fzopen;d:diffs;n:bulknew'
n() {
  local nnn_tmp="${XDG_CONFIG_HOME:-$HOME/.config}/nnn/.lastd"
  nnn -adeHo "$@"
  [ -f "$nnn_tmp" ] && { . "$nnn_tmp"; rm -f "$nnn_tmp"; }
}
EOF

# 2. interactive launch
n
# inside nnn:
#   1 2 3 4   switch contexts (tabs)
#   /         filter current view
#   <space>   toggle-select file (build a multi-file selection)
#   p         move selection to current dir
#   P         copy selection to current dir
#   ;p        run preview-tui plugin (side-pane preview in tmux)
#   ;f        fzf-pick anywhere on $PATH and open with $EDITOR
#   q         quit (and cd into the chosen dir thanks to `n` wrapper)

# 3. one-shot batch rename of selection via $EDITOR
#    (select files with <space>, then ;n -> opens vim/nvim with one
#     filename per line; edit, save, nnn renames atomically)

# 4. detail mode + disk-usage view (find what is eating /var)
nnn -d /var       # detail view
# then press M to switch to disk-usage mode (du-style sizes per entry)
```

## Why This, Not Another

1. **Resource floor is the lowest in the catalog.** `nnn` opens
   in ~10 ms and idles at < 5 MB RAM. On a 4 MB Alpine container,
   on a router's BusyBox shell, on a serial console into an
   embedded box, it is the only TUI file manager that actually
   *runs*. `ranger` / `yazi` / `joshuto` / `xplr` all want
   tens of MB and a Python / Rust / Lua runtime.
2. **Plugins are POSIX shell, not a runtime.** Every plugin is
   a script you can `cat`. There is no embedded interpreter to
   sandbox-escape, no plugin manager that fetches arbitrary code
   on first run, no version skew between the plugin API and the
   host. Writing a plugin is "drop a script in
   `~/.config/nnn/plugins/`" — it gets the selection on stdin,
   `$PWD` in env, and that's the whole API.
3. **Four contexts, not infinite tabs.** Sounds like a limit;
   it is a feature. Four `1234` keys map to four parallel
   directory states, which covers the staging / source /
   destination / scratch workflow without growing into a tab
   bar you have to manage. Less UI, more muscle memory.

For an LLM-CLI workflow, `nnn`'s `-p -` flag prints the user
selection to stdout — pipe it straight into
[`files-to-prompt`](../files-to-prompt/) /
[`repomix`](../repomix/) /
[`code2prompt`](../code2prompt/) without writing a wrapper.

## Vs Already Cataloged

- **Vs [`yazi`](../yazi/):** `yazi` is the modern, Rust, async,
  image-preview-built-in answer. `nnn` is the minimal C answer.
  Pick `yazi` on a workstation where you want pictures and Lua;
  pick `nnn` on a server / container / SSH session where every
  MB matters and POSIX shell is the only runtime you trust.
- **Vs [`joshuto`](../joshuto/):** `joshuto` is the ranger-faithful
  Rust port (Miller columns, TOML config). `nnn` is the
  single-pane minimal answer. `joshuto` looks more like ranger;
  `nnn` is faster to launch and ~10× smaller on disk.
- **Vs [`broot`](../broot/):** orthogonal. `broot` is a fuzzy
  tree-finder optimised for "find the right directory in 200 ms";
  `nnn` is a full file manager optimised for "browse, select,
  move, batch-rename." Use `broot` to *get* somewhere; use `nnn`
  once you are there.
- **Vs [`xplr`](../xplr/):** both target the minimal-TUI niche.
  `xplr` makes every keypress a Lua message (infinite hackability
  via Lua); `nnn` makes every plugin a shell script (infinite
  hackability via POSIX). Pick `xplr` if you want to *script the
  file manager*, `nnn` if you want to *script around it*.

## Caveats

- **Single-pane UI is a learning curve for ranger users.** No
  parent-dir column, no preview pane by default — you toggle
  modes (`d` for detail, `M` for disk-usage) instead. The
  `preview-tui` plugin gets you a side preview but requires
  `tmux`. If you want Miller columns out of the box, prefer
  `joshuto` / `yazi`.
- **Image preview is plugin-only and terminal-dependent.** `nnn`
  itself draws no images; the `preview-tui` / `nuke` plugins
  shell out to `chafa` / kitty's `icat` / `ueberzug` / `viu` and
  inherit those tools' quirks. Stock SSH session = text only.
- **Config is environment variables, not a file.** `nnn` is
  configured via `NNN_*` env vars (`NNN_PLUG`, `NNN_BMS`,
  `NNN_FIFO`, `NNN_OPENER`, ~30 in total). The wins:
  reproducible from `~/.zshrc`, no config-file parsing bugs,
  trivial to ship across machines. The cost: discovering the
  full env-var surface means reading the man page, not a
  commented `config.toml`.
- **Bookmarks / history live in `NNN_BMS` and a binary file.**
  Editing bookmarks means editing the env var; history is in
  `~/.config/nnn/.history` as a binary list, not human-readable.
- **No undo.** `d` (delete) is permanent unless you have
  `NNN_TRASH=1` set (which routes through `trash-cli` /
  `gio trash`). Set it on day one, especially before reaching
  for `nnn` on a production box.
