# joshuto

> **A ranger-clone TUI file manager written in Rust — three-pane
> Miller-columns view (parent dir / current dir / preview), vim-style
> keybinds, async directory loading, and a small TOML config surface
> instead of ranger's Python plugin tax.** Pinned to **v0.9.9**,
> [LICENSE](https://github.com/kamiyaa/joshuto/blob/main/LICENSE),
> LGPL-3.0.

Source: <https://github.com/kamiyaa/joshuto>

## TL;DR

`joshuto` is what you reach for when you want `ranger`'s muscle
memory without dragging Python 3, the `ueberzug` image-preview
sidecar, and the `~/.config/ranger/` plugin sprawl onto every box
you SSH into. A single ~6 MB Rust binary gives you the same
three-column Miller layout, the same `hjkl` / `gg` / `G` / `dd` /
`yy` / `pp` keymap, and the same "preview the highlighted file in
the right pane" workflow, with async I/O so a `cd` into a 50 k-entry
directory does not freeze the UI. Configuration is four TOML files
(`joshuto.toml`, `keymap.toml`, `mimetype.toml`, `theme.toml`) — no
Python config, no plugin manager, no per-CLI lock files. Image
preview piggybacks on whatever your terminal supports (kitty's
graphics protocol, sixel, `chafa`) instead of bundling a
display-server hack. The cost is a smaller feature surface than
mature ranger (no built-in tagging, no archive mounting), but for
the 90 % case — "open a TUI, browse a tree, tab into a few dirs,
yank-and-paste, shell-out" — it is faster and more portable.

## Install

```bash
# Homebrew (macOS / Linux)
brew install joshuto

# Cargo (any platform with a Rust toolchain)
cargo install --locked --git https://github.com/kamiyaa/joshuto.git --tag v0.9.9

# Arch (AUR):  yay -S joshuto
# Nix:         nix-env -iA nixpkgs.joshuto
# Pre-built binaries: https://github.com/kamiyaa/joshuto/releases

# verify
joshuto --version    # joshuto 0.9.9
```

## License

LGPL-3.0 — see
[LICENSE](https://github.com/kamiyaa/joshuto/blob/main/LICENSE).
The CLI binary is freely redistributable; LGPL terms apply if you
link the `joshuto` Rust crates as a library and ship a derivative.

## One Concrete Example

```bash
# 1. launch in $PWD
joshuto

# 2. drop into a directory and have the parent shell `cd` to it on exit
#    (joshuto writes the chosen path to a file, the wrapper sources it)
cat >> ~/.zshrc <<'EOF'
function j() {
  local f="$(mktemp)"
  joshuto --output-file "$f" "$@"
  local d
  [ -f "$f" ] && d="$(cat "$f")" && rm -f "$f"
  [ -n "$d" ] && [ -d "$d" ] && cd "$d"
}
EOF

# 3. minimal keymap override — make `q` exit, `<space>` toggle select
mkdir -p ~/.config/joshuto
cat > ~/.config/joshuto/keymap.toml <<'EOF'
[default_view]
keymap = [
  { keys = [ "q" ],     command = "quit --output-selected-files" },
  { keys = [ " " ],     command = "toggle_visual" },
  { keys = [ "g", "h" ], command = "cd ~" },
  { keys = [ "g", "/" ], command = "cd /" },
]
EOF

# 4. wire up image preview via chafa (works in any 256-color terminal)
cat > ~/.config/joshuto/joshuto.toml <<'EOF'
[preview]
preview_script = "~/.config/joshuto/preview.sh"
preview_shown_hook_script = ""
EOF
```

## Niche It Fills

**A keyboard-first TUI file manager that ships as one static
binary.** The space is well-trodden — `ranger`, `nnn`, `lf`,
`vifm`, `mc`, `yazi`, `xplr`, `broot` — and `joshuto` sits in the
specific corner labelled *"ranger keybinds, no Python, async I/O,
TOML config."* `nnn` is faster and smaller but its keymap is its
own thing; `lf` is closer in spirit but ships fewer batteries
(no built-in image preview wiring); `yazi` is the modern showpiece
with async + plugins but is younger and bigger. Pick `joshuto`
when you already have ranger muscle memory and want to delete the
`pip install ranger` from your bootstrap script.

## Why use it

Three things `joshuto` does that pay off immediately:

1. **Same keymap as ranger, no Python runtime.** `hjkl` to
   navigate, `gg`/`G` to jump, `yy`/`dd`/`pp` to yank/cut/paste,
   `:` for command mode, `/` for search. Drop the binary on a
   fresh box and your fingers already work — no `pip install`,
   no `~/.config/ranger/plugins/` to rsync.
2. **Async directory loading.** `cd` into a directory with 100 k
   files and the UI does not block: the directory listing
   streams in while you scroll, file metadata (size, mtime,
   permissions) populates lazily. Ranger's threaded loader is
   bolt-on; here it is the default.
3. **Composes with the rest of the catalog.** `joshuto
   --output-file` writes the cursor path on exit — pipe that into
   [`bat`](../bat/), [`nvim`](https://neovim.io/), or a one-shot
   [`fd`](../fd/) `+ rg` pipeline without writing a wrapper.
   Pair with [`zoxide`](../zoxide/) (`z foo` to jump, `j` to
   browse) for the canonical "fast jump + visual confirm" loop.

For an LLM-CLI workflow, `joshuto` is the right tool to *visually*
stage which files to feed into [`files-to-prompt`](../files-to-prompt/)
or [`repomix`](../repomix/) — toggle-select a subtree, exit with
the selection list to stdout, pipe straight into the packer.

## Vs Already Cataloged

- **Vs [`yazi`](../yazi/):** both are Rust, both do async I/O,
  both target the post-ranger crowd. `yazi` has a richer plugin
  surface (Lua), built-in tabs, and image preview that "just
  works" via its own protocol layer; `joshuto` is the smaller,
  ranger-faithful, no-plugin-runtime alternative. Pick `yazi`
  for power, `joshuto` for "ranger but Rust."
- **Vs [`broot`](../broot/):** orthogonal. `broot` is a *fuzzy
  tree-finder* (one collapsing tree, `cd` and `:e` shortcuts);
  `joshuto` is a *full TUI file manager* (Miller columns, paste
  buffer, mode lines). Use `broot` for "find the right
  directory in 200 ms," `joshuto` for "browse, select, and
  manipulate files in a session."
- **Vs [`xplr`](../xplr/):** both are Rust TUI file managers;
  `xplr` is hackable-by-Lua (every keypress is a Lua message),
  `joshuto` is hackable-by-TOML (declarative keymaps, no
  scripting runtime). Pick `xplr` when you want to script the
  file manager itself, `joshuto` when you want a finished tool.

## Caveats

- **Smaller feature set than ranger / yazi.** No built-in
  tagging, no archive mounting (`atool`-style), no per-mime
  command menu beyond what you wire into `mimetype.toml`. Power
  users coming from a heavily-customised `~/.config/ranger/`
  will notice missing pieces.
- **Image preview is not bundled.** You wire it through
  `preview_script` to `chafa` / `viu` / kitty's `icat` /
  terminal-specific helpers; there is no "just works" image
  layer the way `yazi` ships one. On a stock SSH session into a
  bare Linux box, expect text previews only.
- **LGPL-3.0, not MIT/Apache.** Permissive enough to redistribute
  the binary and bundle it in package managers, but if you fork
  the codebase into a closed-source product you owe LGPL
  obligations on the modified `joshuto` portion. Most users will
  never hit this; vendor-shipping teams should read the license.
- **Config is four TOML files, not one.** `joshuto.toml`,
  `keymap.toml`, `mimetype.toml`, `theme.toml` all live under
  `~/.config/joshuto/`. The split is principled but the first-time
  user reaching for "where do I rebind a key?" will guess wrong
  once or twice — the answer is `keymap.toml`.
- **Project velocity is one-maintainer slow.** Releases land
  every few months, not every few weeks; bug reports are
  triaged but feature PRs queue. Fine for a stable file
  manager, frustrating if you want bleeding-edge.
