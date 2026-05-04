# cpufetch

> **CPU architecture fetch tool with vendor-logo ASCII art** —
> a single C binary that reads the CPU's identification surface
> (`cpuid` on x86 / x86_64, `mrs` system registers on ARM /
> Apple Silicon, `/proc/cpuinfo` + device tree on RISC-V /
> PowerPC) and prints microarchitecture, manufacturing process
> node (nm), per-core / per-die layout (P-cores + E-cores on
> hybrid Intel / Apple Silicon, CCD count on AMD Zen), L1d /
> L1i / L2 / L3 cache sizes, AVX / AVX-512 / NEON / SVE / AMX
> feature flags, base + boost clocks, and an estimated peak
> single-precision GFLOPS — next to a colourised ASCII logo of
> the vendor (Intel blue, AMD red, Apple grey, ARM teal). Pinned
> to **v1.07** (commit
> `284faf84c3658d0131e0f3784c0dbe05e6ec0ce1`,
> [LICENSE](https://github.com/Dr-Noob/cpufetch/blob/master/LICENSE),
> GPL-2.0-only).

Source: <https://github.com/Dr-Noob/cpufetch>

## TL;DR

`cpufetch` is the right tool when the question is "what *exactly*
is this CPU?" and `lscpu` / `sysctl -a | grep machdep.cpu` /
`cat /proc/cpuinfo` are too verbose, miss microarchitecture
naming (lscpu prints "Intel(R) Core(TM) i7-13700K" but not
"Raptor Lake, 8 P-cores Raptor Cove + 8 E-cores Gracemont, 10nm
Intel 7"), and Geekbench / CPU-Z are GUI-only.

`cpufetch` reads the cpuid leaves directly, decodes the
microarchitecture from the family / model / stepping triple
against a maintained internal table (every Intel µarch since
Nehalem, every AMD since K10, every Apple Silicon since M1,
every recent ARM Cortex / Neoverse), and prints a one-screen
summary suitable for benchmarking notes, CPU procurement
spreadsheets, or "what cloud instance did the CI runner
actually give me" debugging.

```
$ cpufetch
                              ###############
                              #### Intel ####
                              ###############
                Name:         Intel(R) Core(TM) i7-13700K
                Microarchitecture: Raptor Lake
                Technology:   10nm (Intel 7)
                Max Frequency: 5.400 GHz
                Cores:        8 P-cores + 8 E-cores (24 threads)
                AVX:          AVX2,AVX-VNNI
                FMA:          FMA3
                L1i Size:     32KB (8-way) (256KB Total)
                L1d Size:     48KB (12-way) (384KB Total)
                L2 Size:      2MB (16-way) (16MB Total)
                L3 Size:      30MB (12-way)
                Peak Performance: 1612.80 GFLOP/s
```

## Install

```bash
# Homebrew (macOS / Linux)
brew install cpufetch

# Arch Linux
sudo pacman -S cpufetch

# Debian / Ubuntu
sudo apt install cpufetch

# Fedora
sudo dnf install cpufetch

# Build from source
git clone --branch v1.07 https://github.com/Dr-Noob/cpufetch
cd cpufetch
make
sudo make install

# verify
cpufetch --version    # cpufetch v1.07
```

No runtime dependencies; it is one C binary that reads
architecture-specific instructions and `/proc` / `sysctl`.

## License

GPL-2.0-only — see
[LICENSE](https://github.com/Dr-Noob/cpufetch/blob/master/LICENSE).
Binary use is unrestricted; source modifications must be
redistributed under GPL-2.0.

## Hot flags

`cpufetch` is one-shot output, no TUI. Useful flags:

- `--logo-short` / `-l short` — compact single-column logo for
  narrow terminals
- `--logo-long` / `-l long` — wide three-column logo (default
  on a wide terminal)
- `--no-logo` — text only, no ASCII art (for piping into
  scripts)
- `--no-color` — disable ANSI colour (for log files / terminals
  without colour)
- `--style fancy|retro|legacy|invisible` — preset style packs
- `--debug` — dump the raw cpuid leaves / mrs registers used to
  derive the report (the right output to attach to a "cpufetch
  misidentified my CPU" bug report)
- `--verbose` — extra detail (peak DP GFLOPS, peak memory
  bandwidth estimate, full cache associativity breakdown)
- `--raw` — machine-readable single-line output for scripts

## Why use it

- **Microarchitecture is named, not just the SKU.** `lscpu` says
  "Intel(R) Xeon(R) Platinum 8370C CPU @ 2.80GHz"; cpufetch adds
  "Ice Lake-SP, 10nm SuperFin." When the procurement spreadsheet
  needs the µarch and the cloud instance type does not advertise
  it, cpufetch is the source.
- **Hybrid topology rendering.** Intel 12th-gen+ (Alder Lake,
  Raptor Lake, Meteor Lake) and Apple Silicon (M1 / M2 / M3 /
  M4) ship with P-core + E-core asymmetric layouts where `htop`
  shows you N threads but does not say which are which. cpufetch
  prints "8 P-cores + 8 E-cores (24 threads)" — the right
  framing for tuning thread-affinity in a benchmark or a CI
  matrix.
- **AVX-512 / AMX / SVE feature detection** is one column.
  Useful when a binary's perf profile depends on the host
  having a specific SIMD ISA that the cloud-provider docs are
  vague about.
- **Peak GFLOPS estimate** — `cores × FMA-width × FMA-per-cycle
  × clock` derived per-µarch — gives a back-of-envelope ceiling
  for "how much room is there before this workload is
  arithmetic-bound." Not a benchmark; an upper bound.
- **Cross-architecture support.** Same binary on x86_64
  (`cpuid`), aarch64 (`mrs` registers + Apple proprietary
  fields for M-series), RISC-V (`/proc/cpuinfo` + ISA string),
  PowerPC. The cross-platform "tell me about the CPU" tool.

## Vs Already Cataloged

- **Vs [`fastfetch`](../fastfetch/) / [`onefetch`](../onefetch/) /
  [`macchina`](../macchina/):** orthogonal — those are *system*
  fetch tools (distro / kernel / shell / DE / uptime / packages
  + a small CPU line). `cpufetch` is *only* the CPU but with
  microarchitecture-level depth those tools do not have.
  Compose: fastfetch on shell login for the system glance,
  cpufetch when you need the CPU detail for a benchmark note.
- **Vs [`bottom`](../bottom/) / [`btop`](../btop/) /
  [`htop`](../htop/):** orthogonal — those are live monitors
  (CPU utilisation per-core over time); cpufetch is a one-shot
  identification report. Use both: cpufetch once to know what
  the CPU *is*, btop to watch what it *does*.
- **Vs `lscpu` / `dmidecode` / `cat /proc/cpuinfo`:** cpufetch
  is the prettier, microarchitecture-aware, cross-platform
  superset for human reading. Pick the system tools when
  scripting against stable output formats; pick cpufetch when
  the consumer is a human or the report goes into a
  benchmarking spreadsheet.

## Caveats

- **GPL-2.0-only**, not -or-later. Binary redistribution is
  fine; source modification + redistribution carries GPL-2.0
  obligations specifically (no automatic upgrade to GPL-3.0).
- **Microarchitecture identification depends on a maintained
  internal table.** A CPU released after the cpufetch version
  you have installed may show "Unknown microarchitecture" or
  fall back to the family-level name. Update cpufetch annually
  if you work with newest-gen hardware.
- **GPU is not in scope.** A separate sibling project,
  [`gpufetch`](https://github.com/Dr-Noob/gpufetch), covers
  NVIDIA / AMD / Intel discrete + integrated GPUs with the same
  ASCII-logo + microarch report format. Not yet packaged in
  most distros.
- **Peak GFLOPS is theoretical**, not measured. It assumes
  perfect FMA throughput at max boost on every core
  simultaneously — actual sustained workloads hit thermal /
  power / memory-bandwidth limits well below this number.
  Treat as a ceiling, not a benchmark result.
- **Cloud / virtualised CPUs sometimes report partial
  information.** AWS / GCP / Azure VMs expose cpuid leaves but
  may mask topology details (P/E split, cache hierarchy). The
  microarchitecture name is usually still correct; cache and
  core-layout numbers may show "(estimated)" or fall back to
  per-vCPU values.
- **No JSON output today.** `--raw` is a single-line text
  format; for structured ingestion you parse the text or use
  `lscpu --json` instead. Open feature request upstream.
