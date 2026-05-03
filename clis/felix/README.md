# felix

> **A keyboard-driven TUI file manager in a single Rust binary**
> — vim-style modal navigation, register-based yank/cut/paste,
> background image previews via the kitty / sixel graphics
> protocols, on-the-fly archive extraction, and a built-in
> trashcan that survives across sessions, all from one
> dependency-free static executable. Pinned to **v2.16.1**
> ([LICENSE](https://github.com/kyoheiu/felix/blob/main/LICENSE),
> MIT).

Source: <https://github.com/kyoheiu/felix>

## TL;DR

`felix` (binary name `fx`) is a one-pane TUI file manager aimed
at users who want vim ergonomics without the configuration
surface of `ranger` / `lf` / `yazi`. Launch it with `fx`,
navigate with `j`/`k`/`h`/`l`, jump to parent with `h`, enter a
directory with `l`, yank a file with `yy`, paste with `p`,
delete (to trash) with `dd`, search with `/`, sort with `sort`.
A configurable `[exec]` map opens files in the right tool by
extension. Image previews work in any terminal that supports the
kitty graphics protocol (kitty, ghostty, WezTerm, foot) or
sixel.

## Install

```bash
# Homebrew (macOS / Linux)
brew install felix

# Cargo
cargo install --locked felix

# Pre-built binary
# https://github.com/kyoheiu/felix/releases/tag/v2.16.1
curl -L -o felix.tar.gz \
  https://github.com/kyoheiu/felix/releases/download/v2.16.1/felix-x86_64-apple-darwin.tar.gz
tar xf felix.tar.gz
sudo install fx /usr/local/bin/

# Pacman (Arch)
sudo pacman -S felix-rs

# Nix
nix-env -iA nixpkgs.felix-fm

# verify
fx --version              # felix 2.16.1

# Optional: enable shell-cd-on-exit so leaving fx in a directory
# cd's the parent shell into that directory.
# Add to ~/.zshrc or ~/.bashrc:
fx() {
  local tmp_dir
  tmp_dir=$(mktemp)
  command fx --cd-on-exit "$tmp_dir" "$@"
  if [ -s "$tmp_dir" ]; then cd "$(cat "$tmp_dir")"; fi
  rm -f "$tmp_dir"
}
```

First launch creates `~/.config/felix/config.yaml` with the
default exec map and key bindings; edit to taste.

## License

MIT — see
[LICENSE](https://github.com/kyoheiu/felix/blob/main/LICENSE).
Permissive, redistribute and modify freely; no attribution
required for binary distributions.

## One Concrete Example

```bash
# Launch in $PWD
fx

# Inside fx (single pane, vim-style):
#   j / k         move down / up
#   h / l         parent / enter directory (or open file via [exec])
#   gg / G        top / bottom
#   /             search current directory (incremental)
#   :             command line (cd, q, e <path>, sort, etc.)
#
#   yy            yank current item into the register
#   dd            "delete" (move to felix's trashcan)
#   p             paste yanked / cut items into current directory
#   V             visual select (j/k to extend, then yy / dd / p)
#
#   c             rename current item in-place
#   o             create empty file (prompt for name)
#   O             create new directory
#
#   t             toggle hidden files
#   v             toggle preview pane (text or image)
#   z             show / hide trashcan as a virtual directory
#   u             undo last delete (restore from trashcan)
#
#   :e ~/notes    jump to an absolute path
#   :sort time    sort by modification time
#   :sort size    sort by size
#   ZZ / :q       quit (with cd-on-exit shell wrapper, parent
#                 shell ends up in the last viewed directory)

# From the shell:
fx ~/Downloads          # open felix in a specific directory
fx --log                # write a log file to debug exec / preview
fx --print              # on quit, print the final cwd to stdout
                        # (alternative to the cd-on-exit wrapper)
```

Configure `~/.config/felix/config.yaml` to wire openers per
extension:

```yaml
default: nvim
exec:
  zathura:
    - pdf
    - epub
  mpv:
    - mp4
    - mkv
    - webm
  imv:
    - jpg
    - png
    - gif
match_vim_exit_behavior: true
use_full_width: true
```

## Niche It Fills

**Single-pane, no-config, vim-default TUI file manager.** The
existing space splits into two camps: highly-configurable
multi-pane managers ([`ranger`](../ranger/), [`lf`](../lf/),
[`yazi`](../yazi/), [`xplr`](../xplr/), [`superfile`](../superfile/))
that reward investment in dotfiles, and minimalist single-pane
managers ([`nnn`](../nnn/), [`vifm`](../vifm/),
[`broot`](../broot/)) that often have their own conventions.
`felix` is positioned for users who want vim keys to *just work*
on first launch, with image preview and trashcan as built-ins
rather than plugin contracts. The trade is a smaller plugin
ecosystem than `yazi` and a smaller user base than `ranger`, in
exchange for shipping one binary you can drop on a server and be
productive in 30 seconds.

## Why use it

Three things `felix` does that justify another file manager:

1. **Built-in trashcan with `u` undo.** `dd` doesn't really
   delete — it moves to felix's own trash directory under
   `~/.local/share/felix/Trash/`, which the `z` key shows as a
   virtual folder you can navigate, restore from, or empty.
   Forgiving by default; safer than `rm`-bound managers.
2. **Image previews in graphics-capable terminals, no extra
   binary.** Inside kitty / ghostty / WezTerm / foot, `v` on an
   image renders the actual pixels in the preview pane via the
   kitty graphics or sixel protocol. No `chafa`/`viu`
   subprocess to install, no fallback character-art for users
   on those terminals.
3. **One static Rust binary, zero plugin contract.** `cargo
   install --locked felix` and you have everything: navigation,
   trash, search, image preview, archive view. Compare to
   `ranger` (Python + `rifle` openers + previewer scripts) or
   `yazi` (excellent, but bigger surface and a Lua plugin
   manager). On a server where you don't want to pin a Python
   interpreter and twelve openers, `felix` is the simpler drop.

For an LLM-CLI workflow where the agent needs the human to
visually verify a generated screenshot or PDF before approving
the next step, `fx ./out && press v` previews in the terminal
without a desktop session.

## Vs Already Cataloged

- **Vs [`yazi`](../yazi/):** the closest peer. `yazi` is
  feature-richer (multi-tab, async I/O on previews, Lua plugin
  ecosystem, more polished UI) and has a larger user base; pick
  `yazi` if you want a long-term primary file manager and don't
  mind a Lua config. Pick `felix` for a smaller binary, simpler
  config (YAML, one file), and the built-in trashcan-with-undo.
- **Vs [`ranger`](../ranger/):** ranger is the elder
  multi-pane Python file manager with the deepest plugin
  ecosystem and the widest preview support, but it's a Python
  install with `rifle` opener configs and external previewer
  scripts. `felix` is a single Rust binary; trade flexibility
  for portability.
- **Vs [`lf`](../lf/):** `lf` is the Go single-binary alternative
  to ranger, also single-pane by default with explicit
  configuration. `lf` has a richer config DSL (commands,
  remappings, custom previewers) and a longer track record;
  `felix` ships more out-of-the-box (image preview, trashcan)
  with less to configure.
- **Vs [`nnn`](../nnn/):** `nnn` is the C-binary minimalist with
  plugins as shell scripts. Faster startup than felix, but
  steeper key conventions (not vim-default) and previews require
  a separate plugin pipeline. Pick `nnn` for absolute speed,
  `felix` for vim defaults + built-in previews.
- **Vs [`xplr`](../xplr/):** `xplr` is the Lua-scriptable
  power-user file manager — extreme flexibility, almost a
  framework. `felix` is the deliberate "no scripting required"
  alternative for users who don't want to maintain a Lua
  config.
- **Vs [`superfile`](../superfile/):** `superfile` leans on
  multi-pane modern aesthetics (sidebar, pinned dirs); `felix`
  is single-pane and stricter about vim ergonomics.

## Caveats

- **Single-pane only.** No two-pane copy/move workflow. If your
  muscle memory is "left pane source, right pane destination",
  use `yazi`, `ranger`, or `superfile`.
- **Image preview requires a graphics-capable terminal.** kitty,
  ghostty, WezTerm, foot work; classic xterm, GNOME Terminal,
  Apple Terminal don't. Preview falls back to a placeholder.
- **Trashcan is felix-private.** `dd` moves files into
  `~/.local/share/felix/Trash/`, *not* the desktop environment's
  shared trash (`~/.local/share/Trash/` per the FreeDesktop
  spec). Files trashed in `felix` won't appear in the GNOME /
  KDE trash GUI, and vice-versa. If you want desktop-shared
  trash, layer [`trashy`](../trashy/) or [`gomi`](../gomi/) on
  top.
- **No remote / SFTP browsing.** Local filesystem only. For
  remote, use [`termscp`](../termscp/).
- **Smaller plugin ecosystem than yazi / ranger.** YAML config
  for openers and a few hooks; no scripting language. Power
  users who want custom commands per file type may outgrow it.
- **Project pace.** Active maintenance under a single primary
  author. Surface is small enough that this is fine, but don't
  expect a sprint backlog of new features — the trade for a
  small dependency surface.
