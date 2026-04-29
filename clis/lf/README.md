# lf

> **A Go-written single-binary terminal file manager modelled on
> ranger's keymap and Miller-columns layout, minus the Python
> runtime and plugin system. ~3 MB static binary, async file I/O,
> a tiny config DSL (`lfrc`) plus shell-out for everything else.**
> Pinned to **v41** (homebrew),
> [LICENSE](https://github.com/gokcehan/lf/blob/master/LICENSE),
> MIT.

Source: <https://github.com/gokcehan/lf>

## TL;DR

`lf` ("list files") is the minimalist Go answer to ranger: same
Miller-columns layout (parent / current / preview), same `hjkl`
movement, same `yy` / `dd` / `pp` / `:` / `/` muscle memory, but
shipped as one ~3 MB static binary with a 200-line config DSL
(`~/.config/lf/lfrc`) instead of a Python plugin tree. There is
no built-in image preview, no built-in archive mounting, no
built-in `rifle.conf` — `lf`'s philosophy is "shell out for
everything." File previews come from a user-supplied `previewer`
script (typically `bat` for text, `chafa` for images, `ouch list`
for archives), file actions come from `cmd open` definitions in
`lfrc`. Async I/O means a `cd` into a 100 k-entry directory does
not block the UI. The result: ranger ergonomics on a busybox
container, an embedded Linux box, or any host where you do not
want to install Python.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lf         # currently v41

# Pre-built binaries (every platform)
# https://github.com/gokcehan/lf/releases/tag/r41

# Go install (any platform with a Go toolchain)
go install github.com/gokcehan/lf@latest

# Verify
lf -version             # r41
```

## License

MIT — see
[LICENSE](https://github.com/gokcehan/lf/blob/master/LICENSE).
Permissive: redistribute, vendor, fork, ship inside a commercial
product without copyleft obligations.

## One Concrete Example

```bash
# 1. wrapper that cds the parent shell to the directory you exit on
cat >> ~/.zshrc <<'EOF'
function lfcd() {
  local tmp="$(mktemp)"
  lf -last-dir-path="$tmp" "$@"
  if [ -f "$tmp" ]; then
    local dir="$(cat "$tmp")"
    rm -f "$tmp"
    [ -d "$dir" ] && [ "$dir" != "$PWD" ] && cd "$dir"
  fi
}
EOF

# 2. minimal lfrc with previews via bat + chafa, hidden files on
mkdir -p ~/.config/lf
cat > ~/.config/lf/lfrc <<'EOF'
set hidden true
set drawbox true
set icons true
set ratios 1:2:3
set previewer ~/.config/lf/preview
set scrolloff 5

cmd open ${{
    case $(file --mime-type -Lb $f) in
        text/*|application/json) $EDITOR "$f" ;;
        image/*)                 chafa "$f" | less -R ;;
        application/pdf)         pdftotext "$f" - | less ;;
        *)                       xdg-open "$f" >/dev/null 2>&1 & ;;
    esac
}}

cmd mkdir %mkdir -p "$@"
cmd touch %touch "$@"
EOF

# 3. preview script
cat > ~/.config/lf/preview <<'EOF'
#!/bin/sh
case "$(file --mime-type -Lb "$1")" in
  text/*|application/json) bat --color=always --style=plain --paging=never --line-range=:200 "$1" ;;
  image/*)                 chafa --size="$2x$3" "$1" ;;
  application/zip|application/gzip|application/x-tar) ouch list "$1" 2>/dev/null || tar tf "$1" ;;
  *)                       file -b "$1" ;;
esac
EOF
chmod +x ~/.config/lf/preview
```

## Niche It Fills

**The "ranger keymap, no runtime, no plugins" file manager.**
Same family as `joshuto`, `nnn`, `vifm`, `xplr`, `yazi` — `lf`'s
specific corner is *Go static binary + shell-out for everything*.
Pick when you want the smallest dependency footprint that still
gives you ranger ergonomics, and you are happy to wire previews
and openers from POSIX `sh` rather than from a config DSL.

## Why use it

Three things `lf` does that pay off immediately:

1. **One binary, zero runtime.** Drop the ~3 MB binary onto an
   Alpine container, a Raspberry Pi, a remote SSH box — no
   Python, no Lua, no Node. `scp lf` and you have ranger
   ergonomics.
2. **`cmd` definitions are shell.** Every action (`open`,
   `delete`, `extract`, custom verbs) is a `%`-prefixed (sync)
   or `$`-prefixed (interactive) shell snippet in `lfrc`. No
   embedded scripting language to learn — if you can write
   `bash`, you can extend `lf`.
3. **Async by default.** Directory listing, preview generation,
   file operations all run off the UI thread; navigating a
   100 k-entry directory or previewing a 500 MB log does not
   freeze the cursor.

For an LLM-CLI workflow, `lf` is the right tool to *visually*
stage paths on a remote container before piping into
[`files-to-prompt`](../files-to-prompt/) or
[`code2prompt`](../code2prompt/) — toggle-select with `<space>`,
exit with `-print-selection`, pipe straight into the packer.

## Vs Already Cataloged

- **Vs [`ranger`](../ranger/):** `lf` is what you reach for when
  the Python dependency is the problem. Same keymap, smaller
  surface (no `rifle.conf`, no plugin system) — you wire opener
  logic in `lfrc` shell snippets instead.
- **Vs [`joshuto`](../joshuto/):** both are static-binary
  ranger-clones (Go vs Rust). `joshuto` keeps configuration in
  TOML files, `lf` keeps it in a shell-flavored DSL. Pick
  `joshuto` for declarative config, `lf` for "everything is a
  shell snippet" extensibility.
- **Vs [`nnn`](../nnn/):** `nnn` is C, smaller still, with its
  own non-vim keymap and a plugin model based on POSIX scripts
  in `~/.config/nnn/plugins/`. `lf` keeps ranger keys and a more
  configurable opener pipeline. Pick `nnn` for absolute minimal,
  `lf` for "ranger but Go."
- **Vs [`yazi`](../yazi/):** `yazi` is the modern, async, Lua-
  pluggable Rust showpiece with built-in image preview. `lf`
  trades that for "one Go binary, BYO preview script." Pick
  `yazi` for batteries, `lf` for footprint.

## Caveats

- **No built-in image preview.** Wire it via `previewer`
  pointing at `chafa` / `viu` / `kitty +kitten icat`. On a stock
  SSH session into a bare Linux box, expect text-only previews
  unless you install one of those.
- **No built-in archive support.** `extract` / `mount` are
  `cmd`-defined shell snippets you write yourself (`ouch
  decompress` is the easy default; see the upstream wiki for
  patterns).
- **Config DSL is small but quirky.** `set`, `map`, `cmd`, and
  shell-snippet prefixes (`%` / `$` / `!` / `&`) cover the
  whole surface — power users coming from ranger's Python
  `commands.py` will find some things harder to express (custom
  prompt UI, complex multi-step verbs).
- **Single-pane focus mode missing.** `lf` is committed to the
  three-column Miller layout; if you want a single-column tree
  view, look at [`broot`](../broot/) instead.
- **Versioning by date-letter.** Releases are named `r28`, `r33`,
  `r41` etc., not semver — packagers smooth this to `41` /
  `0.41`. Pin by tag (`r41`) when reproducibility matters.
