# upx

> **Ultimate Packer for eXecutables**: a single static binary that
> compresses ELF / Mach-O / PE / DOS executables in place, prepending
> a tiny LZMA / NRV2B / NRV2D / NRV2E decompressor stub so the
> resulting file runs identically with no runtime dependency on any
> external library and no extraction-to-disk step. Pinned to
> **v5.1.1** ([COPYING](https://github.com/upx/upx/blob/devel/COPYING),
> GPL-2.0-or-later with special UPX exception).

Source: <https://github.com/upx/upx>

## TL;DR

`upx` rewrites an executable file as `[stub | compressed-original]`
where the stub is a tiny architecture-specific decompressor that
runs at process start, mmaps the compressed payload, decompresses
it into memory, and jumps to the original entry point. From the
OS's perspective the file is still a normal ELF / Mach-O / PE; from
the user's perspective `./compressed-binary --help` behaves
identically to the original. Typical compression on Go / Rust /
C++ static binaries is **2–4×** (a 70 MB Go binary becomes 18 MB;
a 30 MB Rust binary becomes 9 MB), at a one-time cost of ~50–200 ms
of decompression at process start. Supports ~20 executable formats
across Linux x86_64 / arm64 / arm / mips / riscv64 / ppc64,
Windows x86 / x64, macOS x86_64, and embedded (DOS COM/EXE,
Atari TOS, Watcom LE, etc.). Reversible: `upx -d` restores the
original bit-for-bit.

## Install

```bash
# Homebrew (macOS / Linux)
brew install upx

# Debian / Ubuntu
sudo apt install upx-ucl

# Fedora
sudo dnf install upx

# Arch
sudo pacman -S upx

# Direct binary download (statically linked, no deps)
curl -sSL https://github.com/upx/upx/releases/download/v5.1.1/upx-5.1.1-amd64_linux.tar.xz \
  | tar -xJ --strip-components=1 -C /usr/local/bin upx-5.1.1-amd64_linux/upx

# verify
upx --version    # upx 5.1.1
```

The binary is statically linked, has zero runtime dependencies,
and runs on any kernel ≥2.6 (Linux) / ≥10.13 (macOS) / ≥XP
(Windows). Source builds require CMake + a C++17 compiler.

## License

GPL-2.0-or-later **with the UPX special exception** — see
[COPYING](https://github.com/upx/upx/blob/devel/COPYING). The
exception is the critical detail: it permits redistributing
binaries that have been *packed* with UPX under any licence,
including proprietary ones. Without it, the GPL on the
decompressor stub embedded in every packed file would arguably
infect the packed program itself. With it, packing your Apache-2.0
Go binary or your proprietary product binary with UPX produces a
packed binary you can ship under the original licence, no GPL
copyleft propagation. The exception covers the *embedded stub*;
the UPX *tool* itself remains GPL-2.0+ (matters only if you fork
the tool or embed UPX-the-library, which almost no one does).

## One Concrete Example

```bash
# 1. Default compression (LZMA, level 7 — good speed/ratio balance)
upx myapp
# myapp: 73654272 -> 18923520    25.69%

# 2. Maximum compression (slower at pack time, no runtime difference)
upx --best --lzma myapp

# 3. Brute-force every method and pick the smallest (very slow at pack time)
upx --brute myapp

# 4. Decompress (round-trip restores original bit-for-bit; verify with sha256sum)
upx -d myapp -o myapp.orig
sha256sum myapp.orig original-myapp   # should match if input was unmodified

# 5. Inspect a packed binary without unpacking
upx -l myapp
# File size         Ratio      Format      Name
# 18923520 ->    73654272   25.69%   linux/amd64   myapp

# 6. Test a packed binary's integrity (decompress in memory, discard output)
upx -t myapp

# 7. CI release pipeline: build -> strip -> upx -> publish
go build -ldflags="-s -w" -trimpath -o dist/cli ./cmd/cli
strip dist/cli
upx --best --lzma dist/cli
sha256sum dist/cli > dist/cli.sha256

# 8. Force in-place compression (no .bak), useful in scratch Docker FROM stages
upx --force-overwrite myapp
```

## Niche It Fills

**Shrink a static binary by 2–4× without changing its interface,
its dependencies, or its install path.** Static binaries (Go,
Rust, C++ `--static`) are the right shape for "drop one file in
`/usr/local/bin`, no runtime to install" but they're large — 60 MB
Go binaries and 30 MB Rust binaries are normal because every
dependency is statically linked in. UPX collapses that for the
download / distribution / image-layer-bytes axis without changing
how the binary is invoked, what it depends on at runtime
(nothing — still static), or how `ldd` / `file` see it from the
outside. The cost is ~50–200 ms at first invocation while the
stub decompresses; for batch / one-shot CLIs that's invisible, for
hot-path daemons you opt out.

## Why use it

Three things UPX does that the alternatives do not:

1. **In-place transparent compression — the file is still a normal
   executable.** No wrapper script, no extract-to-tmp step, no
   self-extracting archive that pollutes `/tmp`. The packed file
   has the original's ELF / Mach-O / PE header, the OS loader
   handles it normally, and the embedded stub does the
   decompression in-process before jumping to `_start`. This is
   why UPX-packed binaries work as Docker `ENTRYPOINT`, as
   Kubernetes init-container images, as systemd `ExecStart`, and
   as `cron` targets with no special handling — they look and
   behave exactly like normal binaries to every part of the OS.
2. **Reversible by design.** `upx -d` is not a separate tool, it's
   the same binary with a flag, and it produces a bit-for-bit
   identical original (the SHA-256 round-trips). This matters for
   reproducible builds (rebuild → pack → verify ratio matches),
   for incident forensics (an investigator can `upx -d` a packed
   binary and analyse the original with normal tools), and for
   trust (you can audit what UPX did by unpacking and diffing).
3. **Wide format coverage from one static binary.** ELF (i386,
   amd64, arm, arm64, mips, mipsel, ppc, ppc64, ppc64le, riscv64),
   Mach-O (i386, amd64, arm64), PE (32 / 64), DOS, Atari TOS,
   Watcom LE — 20+ formats, all in one ~600 KB statically-linked
   tool. The right tool for cross-compilation pipelines that emit
   binaries for many architectures from one CI runner.

For container images, UPX on a Go binary turns a "FROM scratch"
image from 73 MB to 19 MB — the layer hash changes by roughly the
delta, the registry stores 4× fewer bytes, and `docker pull` on
the edge node is 4× faster. For Lambda / Cloud Functions cold
starts where the binary download is in the cold-start critical
path, this is directly observable as p99 latency improvement.

## Vs Already Cataloged

- **Vs [`mold`](../mold/):** orthogonal axis. mold is a *linker*
  that produces the binary faster (link time goes from 6 s to
  1 s); UPX shrinks the binary that the linker produced (file size
  goes from 70 MB to 19 MB). They compose: link with mold for
  build-time speed, pack with UPX for distribution-time size.
- **Vs [`slim`](../slim/):** different layer of the stack. slim
  observes a running container and rewrites the *image* to drop
  files the workload didn't touch (typical 20–40× image-size
  reduction). UPX rewrites a *single binary* to compress its
  contents in place (typical 2–4× binary-size reduction). Compose:
  slim the image to remove unneeded files, then UPX the remaining
  binaries inside it. Both reduce bytes shipped, by different
  mechanisms.
- **Vs [`zstd`](../zstd/) / [`xz`](../xz/) (not cataloged) /
  static archive compression:** those compress a file you then
  have to decompress to disk before running. UPX compresses a
  binary that runs *as* the binary, no extraction step. For
  "ship a CLI to users" UPX is the right shape; for "ship a
  tarball that contains many files" zstd is the right shape.
- **Vs `strip` (binutils):** complementary. `strip --strip-all`
  removes debug info and symbol tables (typically 10–30%
  reduction). UPX compresses what's left (additional 50–75%
  reduction on the stripped binary). Always run `strip` *before*
  UPX — the smaller input compresses better and faster. The
  Go-specific equivalent is `-ldflags="-s -w"` at build time.
- **Vs Mach-O `lipo` / fat-binary thinning:** orthogonal. lipo
  removes architectures from a Universal binary (drop arm64 from a
  binary you'll only ship to x86_64 macs). UPX compresses
  whichever arch-slice is left. Compose: lipo first, UPX second.

## Caveats

- **Antivirus and EDR products flag UPX-packed binaries.**
  Historically a large fraction of malware uses UPX (it's free,
  effective, and reverses the obvious "open the file in a hex
  editor and read strings" forensic step) and many AV vendors
  treat "packed with UPX" as a heuristic indicator. **Code-sign
  the packed binary** (Apple Developer ID, Authenticode, Sigstore)
  and submit it to vendors for whitelisting if you're shipping at
  scale. For internal binaries this isn't a problem; for
  consumer-distributed CLIs it can be a deployment headache.
- **Cold-start cost: ~50–200 ms of decompression at process
  start.** Invisible for batch / one-shot CLIs, measurable for
  hot-path serverless / latency-critical daemons. Benchmark before
  packing anything in a startup-latency-sensitive path. UPX's
  `--ultra-brute` doesn't change this cost (it just spends more
  pack-time CPU finding a smaller representation).
- **Memory footprint: the *decompressed* binary is fully resident.**
  UPX decompresses into RAM, not into a memory-mapped file
  representation. A 70 MB binary that the OS would normally
  page-in lazily as you call into different parts of `.text`
  becomes a 70 MB resident allocation immediately. For embedded
  / memory-constrained targets this matters; for desktop / server
  / container targets where RAM is plentiful it does not.
- **Some sandboxing / security tooling rejects packed binaries.**
  systemd `MemoryDenyWriteExecute=yes`, SELinux `execmem`,
  Apple's hardened runtime, and some seccomp profiles object to
  the runtime memory layout transitions UPX's stub performs.
  Test in the target environment; the symptom is "permission
  denied" or "killed: 9" on first run.
- **Reproducibility: the packed output depends on the UPX
  version.** A binary packed with UPX 5.0.0 is not bit-identical
  to the same input packed with 5.1.1, even at identical flags.
  For reproducible-builds pipelines, pin the UPX version in the
  build environment exactly the same way you pin the compiler
  toolchain.
