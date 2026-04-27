# ouch

> **Obvious Unified Compression Helper** — a single Rust binary
> that compresses / decompresses / lists archives with one
> uniform CLI across `tar`, `gz`, `bz`, `bz2`, `bz3`, `xz`,
> `lz4`, `lzma`, `sz` (snappy), `zst`, `zip`, `7z`, and `rar`
> (read-only), inferring format from extension and chaining
> them automatically (`foo.tar.zst` works without flag soup).
> Pinned to **0.7.1** (commit
> `aa7f34fe03ac5a30cc999756ea00e2a5465fe517`,
> [LICENSE](https://github.com/ouch-org/ouch/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ouch-org/ouch>

## TL;DR

`ouch` collapses the muscle memory of `tar -xzf` / `tar -xjf` /
`tar --xz -xf` / `unzip` / `7z x` into three verbs:
`ouch compress` (`c`), `ouch decompress` (`d`), and `ouch list`
(`l`). Format is inferred from the file extension on both sides;
mixed-format chains (`.tar.zst`, `.tar.lz4`, `.tar.bz3`) are
handled in one pipeline without intermediate files. Built-in
overwrite prompts, smart-unpack into subdirs, and a `--threads`
flag for parallel `zstd` / `xz`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ouch

# Linux package managers
# Arch:    pacman -S ouch
# Nix:     nix-env -iA nixpkgs.ouch
# Alpine:  apk add ouch                  (community repo)

# Cargo (any OS with a Rust toolchain)
cargo install --locked ouch

# from a release archive (any OS)
curl -Lo ouch.tar.gz "https://github.com/ouch-org/ouch/releases/download/0.7.1/ouch-aarch64-apple-darwin.tar.gz"
tar xf ouch.tar.gz
sudo install ouch-aarch64-apple-darwin/ouch /usr/local/bin/

# Windows
# scoop install ouch

# verify
ouch --version    # ouch 0.7.1
```

No config file, no daemon — single binary; uses pure-Rust
implementations for most formats (no shelling out to `tar`/`unzip`/`7z`).

## License

MIT — see
[LICENSE](https://github.com/ouch-org/ouch/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. decompress, format inferred from extension
ouch d release.tar.zst
ouch d backup.zip
ouch d firmware.7z
ouch d kernel-source.tar.xz        # chained .tar + .xz handled in one pass

# 2. compress with the format chosen by output extension
ouch c src/ src.tar.zst             # tar then zstd
ouch c notes.md notes.md.gz         # plain gzip on a single file
ouch c photos/ photos.zip           # zip (Windows-friendly)

# 3. list contents without extracting
ouch l artifact.tar.gz
ouch l weird.tar.bz3                # bz3 supported

# 4. extract into a named directory (default: same name as archive)
ouch d release.tar.zst -d /tmp/release/

# 5. quiet / yes-to-all for scripts
ouch d -y release.tar.zst           # auto-overwrite without prompt
ouch d -q release.tar.zst           # suppress per-file output

# 6. parallelise zstd / xz with native threads
ouch c huge-dir/ huge.tar.zst --threads 8

# 7. tune compression level (per-format scale)
ouch c src/ src.tar.zst --level 19  # max zstd
ouch c src/ src.tar.gz  --level 9   # max gzip

# 8. password-protected zip (round-trips with `unzip -P`)
ouch c secrets/ secrets.zip --password 'hunter2'
ouch d secrets.zip --password 'hunter2'

# 9. shell completions
ouch --generate-shell-completion zsh > ~/.zsh/completions/_ouch

# 10. format chain you'd otherwise look up every time
ouch c logs/ logs.tar.bz3            # bzip3 (better ratio than bz2)
ouch c logs/ logs.tar.lz4            # fastest (cold-storage scratch)
```

## Niche It Fills

**Every common archive format has a different CLI.** `tar -xzf`
for `.tar.gz`, `tar -xjf` for `.tar.bz2`, `tar --xz -xf` (or
`-xJf`) for `.tar.xz`, `tar --zstd -xf` for `.tar.zst`, `unzip`
for `.zip`, `7z x` for `.7z`, `unrar x` for `.rar`. Anyone who
deals with archives weekly has a `.zshrc` with a 50-line
`extract()` function that switches on extension. `ouch` ships
that function as a static binary with format autodetection
that's actually right (it sniffs the magic bytes when the
extension lies), plus a corresponding `compress` verb and
`list` verb that work the same way.

## Why use it

Three things `ouch` does that the obvious alternatives do not:

1. **One verb across every format.** `ouch d <anything>` is the
   *only* extract command you need to remember. Format-specific
   flags (`-z`, `-j`, `-J`, `--zstd`) are gone; the extension
   carries the information instead.
2. **Chained formats handled in a single pipeline.** `foo.tar.zst`
   is decompressed by zstd and untarred in one streaming pass
   without writing the intermediate `foo.tar` to disk. Same for
   `.tar.lz4`, `.tar.bz3`, `.tar.xz`. Manual chains
   (`zstd -d foo.tar.zst | tar -x`) are no longer needed.
3. **Smart-unpack avoids the "tarbomb" footgun.** When an
   archive contains many top-level entries, `ouch d` extracts
   into a *subdirectory* named after the archive; when it
   contains a single top-level dir, it extracts directly. Either
   way you don't end up with 200 loose files in your CWD.

For agent / LLM workflows where the model is downloading a
release artifact and doesn't know whether it's `.zip` / `.tar.gz` /
`.tar.zst`, `ouch d <file>` succeeds without the agent having to
branch on the extension and pick a flag set.

## Vs Already Cataloged

- **Vs `tar` / `unzip` / `7z` directly:** the standard tools
  are universally pre-installed and exit-code-stable; they win
  for portable `bash` scripts you ship to unknown environments.
  `ouch` wins for daily interactive use and for scripts you
  control the host of.
- **Vs [`yazi`](../yazi/):** `yazi` (file manager TUI)
  decompresses through its file-action menu and is the right
  tool when you're already navigating in the TUI; `ouch` is the
  right tool when you're at a shell prompt with a `.tar.zst`
  in front of you and just want it gone.
- **Vs `atool` (Perl, well-known):** same idea, different era.
  `atool` shells out to per-format binaries and inherits their
  flag quirks; `ouch` is a single static Rust binary that
  doesn't depend on `unzip` / `xz` / `zstd` being installed.

## Caveats

- **`.rar` is read-only.** `ouch d` and `ouch l` work on rar
  files (via `unrar` Rust bindings), but `ouch c foo.rar` will
  fail — rar compression requires the proprietary encoder. Use
  `.7z` or `.zip` for cross-platform write.
- **Format detection trusts the extension first.** A
  `release.zip` that is actually a `.tar.gz` will be probed
  via magic bytes as a fallback, but archives with no extension
  *and* no recognisable magic header are rejected with an error
  instead of guessed; rename them or pass `--format <fmt>`
  explicitly.
- **Compression levels and parallelism are format-specific.**
  `--level 19` is meaningful for zstd; passing `--level 19` to
  gzip silently clamps to 9. `--threads N` is honoured by zstd
  and xz; ignored by gzip / bzip2. Don't expect uniform
  behaviour from the level / thread flags across formats.
- **No streaming from stdin / to stdout in the common case.**
  `ouch` operates on files (it needs the path / extension).
  For pipe-shaped workflows (`curl ... | tar x`) you still want
  the underlying tool.
- **`.7z` reading uses pure-Rust crates with partial coverage
  of obscure 7z features.** Standard LZMA / LZMA2 archives
  produced by `p7zip` round-trip cleanly; archives using BCJ2
  filters or non-default header encryption may fall back to an
  error. For broken 7z files, fall back to `7z x`.
