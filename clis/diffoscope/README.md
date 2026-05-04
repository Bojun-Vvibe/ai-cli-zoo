# diffoscope

> **Recursive, format-aware "what actually changed between these two
> blobs" tool**: hand it two files, two directories, two `.deb`s, two
> ISOs, two firmware images, or two PDFs and it will unpack them as
> deep as it can with whatever helpers are installed and emit a
> structured diff that tells you the answer instead of "binary files
> differ." Pinned to **v318**
> ([COPYING](https://salsa.debian.org/reproducible-builds/diffoscope/-/blob/main/COPYING),
> GPL-3.0-or-later).

Source: <https://salsa.debian.org/reproducible-builds/diffoscope>
(GitHub mirror: <https://github.com/anthraxx/diffoscope>)

## TL;DR

`diffoscope` is the canonical tool of the
[Reproducible Builds](https://reproducible-builds.org) project for
answering *exactly* what differs between two artifacts. Where
`diff -r` stops at "binary files A and B differ," diffoscope keeps
going: it recognises ~80 file formats (Android APK, Debian `.deb`,
RPM, ISO 9660, squashfs, ext4, tar, zip, jar, ar, cpio, ELF, Mach-O,
PE, `.class`, `.dex`, `.pyc`, PDF, docx, PNG, JPEG, SQLite,
WebAssembly, Git pack files, …), unpacks each one with the right
external helper (`unzip`, `7z`, `binutils`, `pdftotext`,
`apktool`, `xxd`, `radare2`, `disasm` from `binutils`, …), and
recurses into the contents diffing layer by layer. Output is plain
text, ANSI-coloured terminal, HTML (single page or multi-page split
on size), JSON, RestructuredText, or Markdown. Falls back to
fuzzy-matched hexdump when nothing else applies. The whole point
is to localise "this `.deb` rebuild is not bit-identical to the
previous one" down to "the embedded `.so` has a `.note.gnu.build-id`
that differs because `__DATE__` was baked in by `gcc`."

## Install

```bash
# Debian / Ubuntu (where it originated; gets the recommended helper set)
sudo apt install diffoscope

# Fedora / RHEL
sudo dnf install diffoscope

# Arch
sudo pacman -S diffoscope

# macOS (Homebrew)
brew install diffoscope

# pip (minimal — no external helpers; install the ones you need separately)
pip install diffoscope==318

# Docker (everything bundled, ~1.5 GB image, but zero host pollution)
docker run --rm -t -w "$(pwd)" -v "$(pwd):$(pwd):ro" \
  registry.salsa.debian.org/reproducible-builds/diffoscope file1 file2

# verify
diffoscope --version            # diffoscope 318
diffoscope --list-tools         # which external unpackers are reachable
```

The `--list-tools` output is critical: diffoscope is only as deep
as the helpers installed alongside it. The Debian package depends
on the recommended set; pip installs do not. If a recursion stops
at "binary differ," the missing helper is what to install next.

## License

GPL-3.0-or-later — see
[COPYING](https://salsa.debian.org/reproducible-builds/diffoscope/-/blob/main/COPYING).
Strong copyleft. Fine to run as a CI gate, fine to ship inside an
internal pipeline, but if you redistribute a modified diffoscope (or
embed it in a derivative product) you ship the source under GPL-3.0+.
For most uses ("run it, read its output, gate the build") the
licence is invisible — it matters only if you fork.

## One Concrete Example

```bash
# 1. Compare two Debian packages built on different days — find the
#    one byte that broke reproducibility.
diffoscope --html=report.html \
    --max-report-size 100000000 \
    foo_1.2.3-1_amd64.deb foo_1.2.3-1_amd64.rebuild.deb
# report.html opens to: control.tar.xz/control differs (Date field)
#                       data.tar.xz/usr/bin/foo: .note.gnu.build-id differs
#                       data.tar.xz/usr/share/doc/foo/changelog.gz mtime differs

# 2. Compare two firmware images (squashfs inside) byte-by-byte at
#    the file-tree level.
diffoscope --text-color=always firmware-v1.bin firmware-v2.bin | less -R

# 3. Compare two release tarballs and emit JSON for a CI gate that
#    fails on any non-whitelisted change.
diffoscope --json diff.json release-v1.tar.gz release-v2.tar.gz
jq '.. | select(.source1?) | .source1' diff.json   # list every changed inner path

# 4. Compare two directories of build outputs, capping recursion so
#    a 50 GB rootfs doesn't take forever.
diffoscope --max-container-depth 4 --exclude '*.log' \
    build-a/ build-b/

# 5. Compare two APKs (recurses into AndroidManifest.xml, classes.dex,
#    META-INF, resources.arsc, embedded .so).
diffoscope app-v1.0.apk app-v1.1.apk --html-dir=apk-diff/

# 6. Reproducibility CI gate: rebuild and assert bit-identical.
make
cp dist/foo.tar.gz /tmp/build-a.tar.gz
make clean && make
cp dist/foo.tar.gz /tmp/build-b.tar.gz
diffoscope /tmp/build-a.tar.gz /tmp/build-b.tar.gz \
    || { echo "build is not reproducible"; exit 1; }
```

## Niche It Fills

**The "files differ — but how, and why?" tool for any pair of
artifacts more structured than a text file.** `diff` and `cmp` give
you bytes; `xxd | diff` gives you hex; `unzip && diff -r` works for
one zip but not nested zips, not signed APKs, not squashfs-inside-
ISO-inside-tar. diffoscope is the recursive, format-aware engine
that turns "these two `.deb`s differ" into a structured tree showing
exactly which inner file, which inner section, which inner symbol
changed — the level of detail you need to make builds reproducible
or to audit "what actually shipped" between two release artifacts.

## Why use it

Three things diffoscope does that the obvious alternatives do not:

1. **Recursive unpacking with a registry of ~80 formats.** A Debian
   `.deb` is `ar(control.tar.xz, data.tar.xz, debian-binary)`. The
   `data.tar.xz` is xz-compressed tar of the filesystem. Inside the
   filesystem are ELF binaries, Python `.pyc` files, gzipped
   manpages, and PNG icons. diffoscope unpacks all of it and runs
   the appropriate format-specific diff at each layer (`readelf
   --all`, `dis-asm`, `pngmeta`, `gunzip | diff`), so the report is
   "section `.note.gnu.build-id` of `usr/bin/foo` differs," not
   "data.tar.xz differs." No other widely-packaged tool does this
   recursion across this many formats.
2. **Designed for reproducibility verification, not just human
   reading.** Output is structured (JSON, RST, HTML, Markdown) and
   stable enough to diff against itself, so the
   [reproducible-builds tests](https://tests.reproducible-builds.org)
   can mass-process millions of comparisons and bucket the failure
   modes (timestamp embedded by `__DATE__`, non-deterministic file
   order in tar, locale leak via `strftime`, build-path leak via
   `__FILE__`). For a CI gate the JSON output + `jq` filter is the
   correct way to whitelist known-non-reproducible noise (e.g.
   embedded RPM `BuildHost`) without ignoring real regressions.
3. **Falls back gracefully.** When no specific unpacker matches,
   diffoscope falls back to a hexdump diff with fuzzy matching that
   highlights inserted/deleted runs rather than showing every byte
   after a one-byte insertion as different — you still get useful
   localisation on truly opaque blobs (firmware blobs without a
   recognised header, proprietary archive formats).

For supply-chain audits ("did the artifact in the registry actually
come from the source in this commit?"), diffoscope is the second
tool you reach for after [`cosign verify-blob`](../cosign/) — the
signature proves *who* signed it; diffoscope proves *what* is
inside.

## Vs Already Cataloged

- **Vs [`difftastic`](../difftastic/):** different problem domain.
  difftastic does AST-aware diff for *source code* in ~30 languages
  (Tree-sitter parsers) — the right tool when reviewing a code PR
  with reformatting noise. diffoscope does *binary / archive /
  filesystem* diff with format-specific recursion — the right tool
  when comparing two release artifacts, two container layers, two
  packages. Complementary: difftastic for the source diff in the
  PR, diffoscope for the artifact diff in the release.
- **Vs [`delta`](../delta/) and [`riff`](../riff/) (not cataloged
  yet):** those are pretty-printers / interactive viewers for
  unified diffs over text. diffoscope is upstream of any "diff"
  notion at all — it unpacks first, then diffs. You could pipe
  diffoscope's text output into `delta` for terminal styling, but
  that misses the HTML / JSON outputs which are the canonical
  consumption shapes.
- **Vs [`hexyl`](../hexyl/) and [`bingrep`](../bingrep/):** those
  inspect *one* binary (hex dump, ELF section listing). diffoscope
  inspects the *delta* between two binaries (or two archives, or
  two filesystems). Composable: `hexyl` to read what one section
  looks like, diffoscope to know that section changed.
- **Vs `unzip && diff -r` (manual):** hand-rolling the recursion
  works for one nested zip but breaks at the first nested squashfs,
  the first compiled `.pyc`, or the first signed APK whose
  signature block is not byte-identical despite identical contents.
  diffoscope's value is the curated registry of unpackers and
  format-specific renderers built up over a decade of
  reproducible-builds work.
- **Vs [`syft`](../syft/) / [`grype`](../grype/) / [`trivy`](../trivy/):**
  those produce SBOMs and scan for known CVEs in the *contents* of
  an image. diffoscope tells you what *changed* between two images.
  Complementary: syft for "what's in this image," diffoscope for
  "what differs between the image we shipped and the image we built
  this morning."

## Caveats

- **Helpers are mandatory, not optional, for useful output.** A
  bare `pip install diffoscope` gives you the engine but none of
  the unpackers — diffs of `.deb` / `.rpm` / `.apk` / squashfs will
  fall back to "binary differ." Always run `diffoscope --list-tools`
  on the host where it will run and install whatever the relevant
  inputs need (`apktool`, `7zip`, `xz-utils`, `binutils`,
  `squashfs-tools`, `default-jdk` for `.class` disassembly,
  `python3-pdfminer` for PDF text, …). The Debian / Fedora / Arch
  packages do this for you; pip and Homebrew do not.
- **Some helpers run untrusted input through complex parsers.**
  diffoscope shells out to many external tools (`apktool`,
  `radare2`, `binwalk`, `7z`) that have had CVEs over the years.
  Run diffoscope on untrusted inputs inside a sandbox (Docker
  image, `bwrap`, `firejail`) — the upstream Docker image is the
  default-safe shape.
- **HTML reports can be enormous.** A diff of two ~500 MB packages
  with thousands of changed inner files can produce gigabyte HTML.
  Use `--max-report-size`, `--max-page-size` (multi-page split),
  and `--html-dir` (one file per directory level) to keep reports
  navigable, and `--exclude '*.log' --exclude '*.cache'` to drop
  noise paths.
- **Format coverage is broad but not exhaustive.** Proprietary
  archive formats, custom firmware containers, and bespoke
  serialisations will fall through to the hexdump fallback. The
  fix is to either pre-process inputs into a recognised format or
  to write a small helper module — diffoscope's plugin surface is
  Python and the
  [`diffoscope.comparators`](https://salsa.debian.org/reproducible-builds/diffoscope/-/tree/main/diffoscope/comparators)
  tree shows the pattern.
- **Not a replacement for `git diff` on text/source.** For
  source-code review use [`difftastic`](../difftastic/) /
  [`delta`](../delta/). diffoscope's strengths are binaries,
  archives, and filesystems where format-aware unpacking matters.
