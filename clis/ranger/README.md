# ranger

> **The original Python TUI file manager with Miller-columns layout
> (parent / current / preview), vim keybinds, and a long-standing
> plugin ecosystem under `~/.config/ranger/plugins/`. The reference
> implementation that every "Rust ranger-clone" in this catalog
> measures itself against.** Pinned to **v1.9.4**,
> [LICENSE](https://github.com/ranger/ranger/blob/master/LICENSE),
> GPL-3.0-or-later.

Source: <https://github.com/ranger/ranger>

## TL;DR

`ranger` is the canonical three-pane keyboard-driven file manager:
parent dir on the left, current dir in the middle, file or directory
preview on the right. `hjkl` to move, `gg` / `G` / `J` / `K` for
jumps, `yy` / `dd` / `pp` for yank / cut / paste, `:` for command
mode, `/` for fuzzy search, `S` to drop into a shell at the
highlighted path. Configured by Python (`commands.py`,
`rc.conf`, `rifle.conf`, `scope.sh`) so anything you can write in
Python can become a key binding or a custom file-opener rule.
Image preview works in `kitty`, `iTerm2`, `wezterm` (native graphics
protocols) or via `w3mimgdisplay` / `ueberzug` / `chafa` on
Linux X11. The cost is the Python runtime + plugin sprawl that
the Rust clones (`joshuto`, `yazi`, `xplr`, `lf`) explicitly avoid;
the benefit is the most mature config surface and the largest
plugin catalog in the space.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ranger     # currently 1.9.4

# pipx (cross-platform, isolated)
pipx install ranger-fm

# Distro packages also ship it: apt / dnf / pacman / nix
# Verify
ranger --version        # ranger version: 1.9.4
```

## License

GPL-3.0-or-later — see
[LICENSE](https://github.com/ranger/ranger/blob/master/LICENSE).
The binary is freely redistributable; if you fork ranger into a
distributed product the copyleft terms apply.

## One Concrete Example

```bash
# 1. launch in $PWD, copy the cwd to the parent shell on quit
#    (ranger writes the chosen path to a file; the wrapper sources it)
cat >> ~/.zshrc <<'EOF'
function rr() {
  local f="$(mktemp)"
  ranger --choosedir="$f" "$@"
  local d
  [ -f "$f" ] && d="$(cat "$f")" && rm -f "$f"
  [ -n "$d" ] && [ "$d" != "$PWD" ] && cd "$d"
}
EOF

# 2. minimal rc.conf override — wider preview pane, hidden files on
mkdir -p ~/.config/ranger
cat > ~/.config/ranger/rc.conf <<'EOF'
set show_hidden true
set column_ratios 1,2,4
set preview_images true
set preview_images_method kitty
EOF

# 3. open the highlighted file in $EDITOR with one key
cat > ~/.config/ranger/commands.py <<'EOF'
from ranger.api.commands import Command
class edit_in_editor(Command):
    def execute(self):
        import os, subprocess
        subprocess.call([os.environ.get("EDITOR", "vi"), self.fm.thisfile.path])
EOF
# then in rc.conf:  map E console edit_in_editor
```

## Niche It Fills

**The reference Python TUI file manager.** Everything in the
"keyboard-driven, three-pane, vim-keys" family — `joshuto`, `yazi`,
`xplr`, `lf`, `vifm` — defines itself relative to ranger. Pick
`ranger` when you want the most complete out-of-the-box
configuration surface, the largest plugin catalog, and you do not
mind a Python 3 runtime on every box you log into.

## Why use it

Three things `ranger` does that pay off immediately:

1. **`rifle` opener rules.** `~/.config/ranger/rifle.conf` is a
   declarative `mime / ext` → command table — open `.md` with
   `glow`, `.png` with `feh`, `.tar.gz` with `ouch list`, source
   files with `$EDITOR`. The same rules drive `l` / `<Enter>` and
   `r` (open with…), so file-type dispatch is one config file
   instead of per-extension wrappers.
2. **`scope.sh` previews everything.** A single shell script in
   `~/.config/ranger/scope.sh` decides how each file type
   previews — text via [`bat`](../bat/), JSON via `jq`, archives
   via `ouch list`, PDFs via `pdftotext`, images via `chafa` /
   `kitty +kitten icat`. One file, total control over the right
   pane.
3. **Plugins are real Python modules.** Drop a file into
   `~/.config/ranger/plugins/` and it runs at startup with full
   access to the `fm` object — bulk rename via
   [`vidir`](https://joeyh.name/code/moreutils/), git status
   highlighting, devicons, `zoxide` integration. The Rust clones
   have config surfaces; only ranger has a *programmable* one.

For an LLM-CLI workflow, ranger pairs with
[`files-to-prompt`](../files-to-prompt/) and
[`repomix`](../repomix/) — visually stage a subtree, drop into a
shell with `S`, run the packer on the selected paths.

## Vs Already Cataloged

- **Vs [`joshuto`](../joshuto/):** `joshuto` is the Rust port of
  ranger's keymap and Miller-columns layout, minus the Python
  runtime and plugin system. Pick `joshuto` for portability and
  cold-start latency, `ranger` when you need the plugin ecosystem
  or already have a configured `~/.config/ranger/`.
- **Vs [`yazi`](../yazi/):** `yazi` is the modern Rust showpiece
  — async I/O by default, Lua plugins, native image preview
  protocol. Pick `yazi` for new setups; pick `ranger` when you
  want the established Python surface and `rifle` opener rules.
- **Vs [`nnn`](../nnn/):** `nnn` is C, near-zero memory, and uses
  its own non-vim keymap. Orthogonal — `nnn` for "single binary
  on a remote box," `ranger` for "configured workstation."
- **Vs [`broot`](../broot/):** `broot` is a *fuzzy tree finder*
  (one collapsing tree, `cd` shortcuts); `ranger` is a *full
  file manager* (panes, paste buffer, opener rules). Use both.

## Caveats

- **Python 3 dependency on every host.** ~50 MB of CPython on
  disk plus stdlib. Fine on workstations, painful on busybox
  containers and embedded targets (which is exactly why the Rust
  clones exist).
- **GPL-3.0-or-later.** Stronger copyleft than the
  MIT/Apache/LGPL of most clones. Free to use and redistribute;
  vendoring into a closed-source product is restricted.
- **Image preview wiring is per-terminal.**
  `preview_images_method` covers `kitty` / `iterm2` / `wezterm`
  natively but Linux X11 still needs `ueberzug` (now
  `ueberzugpp`) for "real" images in non-graphics-protocol
  terminals — one more moving piece.
- **Release cadence is slow.** v1.9.4 has been the stable tag for
  a long stretch; bug-fix patches land but feature work has
  largely migrated to clones. Stable, not bleeding-edge.
