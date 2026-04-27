# broot

> **A new way to see and navigate directory trees** — a single
> Rust binary that renders the filesystem as a fuzzy-searchable,
> depth-balanced tree TUI with size / git / permissions overlays
> and a built-in command bar that drives `cd`, `rm`, `mv`, `cp`,
> `chmod`, and shell verbs against the highlighted path. Pinned
> to **v1.56.2** (commit
> `a4ca16ef73c3aba931886cc7b89a5f25fc487a47`,
> [LICENSE](https://github.com/Canop/broot/blob/main/LICENSE),
> MIT).

Source: <https://github.com/Canop/broot>

## TL;DR

`broot` (invoked as `br` after the shell-function install) is
a one-screen tree browser that fits any directory — no matter
how deep — into your terminal by collapsing siblings that don't
match your fuzzy pattern. Type a few letters, the tree re-balances
to keep the matches visible with their context; hit `:` (or use
`alt-letter` shortcuts) to run a verb against the highlighted
path; press `enter` to `cd` your shell into a directory or open
a file in `$EDITOR`. Designed to replace the `ls` → `cd` →
`ls` → `cd` loop with a single live view.

## Install

```bash
# Homebrew (macOS / Linux)
brew install broot

# Linux package managers
# Arch:    pacman -S broot
# Debian:  apt install broot          (trixie+)
# Fedora:  dnf install broot
# Nix:     nix-env -iA nixpkgs.broot
# Alpine:  apk add broot              (community repo)

# Cargo (any OS with a Rust toolchain)
cargo install --locked broot

# from a release archive (any OS)
curl -Lo broot.zip "https://github.com/Canop/broot/releases/download/v1.56.2/broot_1.56.2.zip"
unzip broot.zip
sudo install build/x86_64-unknown-linux-musl/broot /usr/local/bin/

# first run installs the `br` shell function (one-time)
broot --install
exec $SHELL

# verify
broot --version    # broot 1.56.2
```

The `br` shell function is what makes `enter`-on-a-directory
actually `cd` your parent shell — without it, `broot` can only
print the path. Run `broot --install` once per shell.

## License

MIT — see
[LICENSE](https://github.com/Canop/broot/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. open the current directory as a tree
br

# 2. fuzzy-filter to anything matching "auth"
br
# then type:  auth
# tree collapses everything that doesn't lead to a match

# 3. show sizes (recursive, computed in background)
br -s

# 4. show git status overlay (M / A / ? markers per file)
br -g

# 5. show permissions + dates (full ls -l style columns)
br -dp

# 6. hidden files + ignored files (overrides .gitignore)
br -hi

# 7. one-shot: print all matches, no TUI (pipeable)
br --cmd ":pp" --no-style src/

# 8. drive verbs from the command bar
br
# then type:  :rm        delete highlighted path (asks confirm)
#             :mv newname rename in place
#             alt-e      open in $EDITOR (default verb)
#             ctrl-→     focus into the highlighted directory
#             ctrl-←     pop back out

# 9. project-scoped config: place .broot/conf.hjson at repo root
#    to define repo-specific verbs and column layouts
```

## Niche It Fills

**`tree` shows you everything; `fzf` shows you matches with no
context; `ranger` is a two-pane file manager that wants the
whole screen.** `broot` sits between them: it always shows the
*tree* (so you keep your sense of where you are), but
fuzzy-filtering collapses unrelated branches so a 50 000-file
repo fits in 40 lines. Then verbs (`:rm`, `:mv`, `alt-e`,
`enter`-to-`cd`) act on the highlighted path without leaving
the view. The result is a navigation primitive that replaces
the `ls` / `cd` / `find` / `xargs` chain with one live screen.

## Why use it

Three things `broot` does that the obvious alternatives do not:

1. **Tree-shaped fuzzy search.** Type `auth` and you see the
   matched files *plus their parent path*, not a flat
   `fzf`-style hit list. The "where in the project is this"
   context is preserved at all times.
2. **Background recursive size computation.** `br -s` annotates
   every directory with its on-disk size, computed lazily as
   the cursor moves; you don't pay the full `du -sh *` cost up
   front, and the answer appears progressively.
3. **`enter` actually changes your shell's directory.** Once
   `br` is installed as a shell function, hitting `enter` on a
   directory `cd`s your *parent* shell into it — the only way
   `cd`-on-select works in a TUI without exec-ing a subshell.

For agent / LLM workflows where the model is exploring an
unfamiliar repo, `broot --cmd ":pp" --no-style` produces a
deterministic, paged tree printout that's easier to feed into
a context window than the recursive output of `ls -R`.

## Vs Already Cataloged

- **Vs [`yazi`](../yazi/):** `yazi` is a full-screen Rust file
  *manager* (two-pane Norton-Commander descendant with image
  preview, async I/O, mouse). `broot` is a one-pane
  *navigator* whose superpower is fuzzy-collapsing a tree and
  driving `cd` / verb actions against it. Use `yazi` when you
  want a persistent file-manager session; use `broot` when you
  want to fly into a directory and get back out.
- **Vs [`fd`](../fd/) + [`fzf`](https://github.com/junegunn/fzf):**
  `fd | fzf` gives you a flat picker. `broot` gives you the same
  fuzzy match *with the tree shape preserved*, plus the verb
  bar to act on the result. For "find one path then `cat` it"
  `fd | fzf` wins; for "explore a directory I don't know"
  `broot` wins.
- **Vs [`eza`](../eza/) (`ls` replacement):** `eza` is a one-shot
  printer (you run it, read it, exit); `broot` is an
  interactive session. Use `eza` in scripts and `cat`-style
  one-liners; use `broot` when you'd otherwise be running `ls`
  three times in a row.
- **Vs [`zoxide`](../zoxide/):** complementary — `zoxide`
  takes you to a directory you've *already* visited (jump by
  frecency); `broot` takes you to a directory you *haven't*,
  by letting you see the tree and pick. Pair: `zoxide` for
  return-trips, `broot` for first-trips.

## Caveats

- **`enter`-as-`cd` requires the shell function.** If you only
  install the binary and skip `broot --install`, `enter` on a
  directory will print the path and exit — surprising at first.
  Re-run `broot --install` after switching shells (bash → zsh
  → fish each get their own hook).
- **Recursive size on huge trees costs disk I/O.** `br -s` on a
  multi-million-file repo or a network mount can keep spinning
  for minutes. The TUI stays responsive (sizes appear
  progressively), but the I/O bill is real; skip `-s` on
  network filesystems unless you mean it.
- **Verbs act immediately on confirmation.** `:rm` asks for a
  `y/n`, but there's no undo afterwards. Treat `broot` like a
  shell, not like a recycle-bin file manager.
- **Custom verbs live in HJSON, not JSON.** `~/.config/broot/conf.hjson`
  uses HJSON (JSON with comments / unquoted keys); editors
  without HJSON syntax can mis-flag it. The config schema is
  documented at <https://dystroy.org/broot/conf_file/>.
- **Image / binary preview is opt-in and depends on Kitty / iTerm
  graphics.** The right pane (`ctrl-p`) renders text by default;
  binary/image preview only lights up under terminals that
  support the Kitty graphics protocol or iTerm inline images.
