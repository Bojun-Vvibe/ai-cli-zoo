# eget

> **Easy pre-built binary installer for GitHub releases** — a
> single Go binary that, given an `owner/repo` slug, queries the
> GitHub Releases API, picks the asset matching the current
> OS/arch, downloads it, verifies checksums (and signatures when
> present), unpacks the archive, and drops the executable into a
> directory on `$PATH`. Pinned to **v1.3.4** (commit
> `19d78fc1eec5d0792b691f5e1b2998c7ae61d323`,
> [LICENSE](https://github.com/zyedidia/eget/blob/master/LICENSE),
> MIT).

Source: <https://github.com/zyedidia/eget>

## TL;DR

`eget` is what you reach for when a tool you want is published
only as a GitHub release tarball — no Homebrew formula, no apt
repo, no `cargo install` story — and you don't want to write a
five-line `curl | tar | install` snippet for every new toy. One
command (`eget zyedidia/micro`) figures out the right asset,
unpacks it, and installs the binary; an optional `~/.eget.toml`
turns the tool into a per-user package manager (`eget --upgrade`
re-checks every pinned tool against latest releases).

## Install

```bash
# Homebrew (macOS / Linux)
brew install eget

# Go (any OS with toolchain)
go install github.com/zyedidia/eget@v1.3.4

# bootstrap script (no package manager)
curl https://zyedidia.github.io/eget.sh | sh
sudo mv eget /usr/local/bin/

# verify
eget --version    # eget version 1.3.4
```

## Sample usage

```bash
# install the latest release of a tool, dropping the binary in $PWD
eget jesseduffield/lazygit

# install into ~/.local/bin instead of cwd
eget --to ~/.local/bin sharkdp/fd

# pin a specific tag
eget --tag v0.35.0 sharkdp/bat

# only download a specific asset name
eget --asset musl ogham/exa

# list assets without downloading (useful for unusual repos)
eget --download-only --asset list cli/cli

# config-driven: list tools in ~/.eget.toml, then upgrade them all
cat <<'EOF' >> ~/.eget.toml
[sharkdp/bat]
[sharkdp/fd]
[junegunn/fzf]
EOF
eget --upgrade
```

## When to pick this

Pick `eget` when you live on a machine where you cannot install
system packages (locked-down workstation, ephemeral CI runner,
fresh VPS without `sudo`), but you can write to `~/.local/bin`
and you want the same one-liner shape (`eget owner/repo`) for
every modern dev tool that ships a GitHub release. Reach for a
real package manager (`brew`, `apt`, `pacman`, `nix`) when you
also want shared libraries, man pages, shell completions, and
auto-update on a schedule — `eget` only manages single static
binaries and runs on demand.

## License

MIT — see [LICENSE](https://github.com/zyedidia/eget/blob/master/LICENSE).
