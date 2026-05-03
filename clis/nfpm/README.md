# nfpm

> **Dependency-free deb / rpm / apk / archlinux / ipk package
> builder.** A single Go binary that reads one declarative
> `nfpm.yaml` and emits native packages for Debian / Ubuntu,
> RHEL / Fedora / Rocky / Alma, Alpine, Arch, and OpenWrt
> targets — no `dpkg-deb`, no `rpmbuild`, no `fakeroot`, no
> Docker matrix per distro. Pinned to **v2.46.3**
> ([LICENSE.md](https://github.com/goreleaser/nfpm/blob/main/LICENSE.md),
> MIT).

Source: <https://github.com/goreleaser/nfpm>

## TL;DR

`nfpm pkg --packager deb --target out/` reads a single YAML
manifest (name, version, maintainer, depends, contents) and
writes a real `.deb` that `dpkg -i` accepts on a clean Debian
install — same manifest re-emits a `.rpm`, `.apk`, `.ipk`, or
Arch `.pkg.tar.zst` by changing the `--packager` flag. Built by
the GoReleaser org as the packaging engine GoReleaser uses
under the hood, so the same toolchain that ships your Go binary
to GitHub Releases also ships the distro packages — and you can
use `nfpm` standalone for any language (Rust, Python, shell,
static assets) where the upstream packaging tooling is heavier
than the actual payload.

## Install

```bash
# Homebrew (macOS / Linux)
brew install nfpm

# Go install
go install github.com/goreleaser/nfpm/v2/cmd/nfpm@latest

# Docker (no host install)
docker run --rm -v "$PWD":/tmp/pkg -w /tmp/pkg \
  goreleaser/nfpm package --config nfpm.yaml \
  --packager deb --target /tmp/pkg/out
```

## Example

```yaml
# nfpm.yaml
name: myapp
arch: amd64
version: v1.4.0
maintainer: Ops <ops@example.com>
description: My app, packaged for every distro from one manifest.
license: Apache-2.0
depends:
  - ca-certificates
contents:
  - src: ./dist/myapp
    dst: /usr/bin/myapp
  - src: ./packaging/myapp.service
    dst: /lib/systemd/system/myapp.service
  - src: ./packaging/config.yaml
    dst: /etc/myapp/config.yaml
    type: config|noreplace
```

```bash
# Same manifest, four distro packages
nfpm pkg --packager deb --target out/
nfpm pkg --packager rpm --target out/
nfpm pkg --packager apk --target out/
nfpm pkg --packager archlinux --target out/
```

## When to use

- You ship a single static binary (Go / Rust / Zig) and want
  native distro packages without learning `dpkg-deb` +
  `rpmbuild` + `apkbuild` + `PKGBUILD` syntax.
- Your CI matrix already produces a Linux binary and you want
  one extra step to drop `.deb` / `.rpm` / `.apk` artifacts on
  every release tag.
- You use GoReleaser and want to add Linux packages — `nfpm`
  is the engine GoReleaser already calls; configuring it
  directly gives you the same output for non-Go projects.

## When NOT to use

- The package needs distro-specific build hooks (e.g. compiling
  a kernel module under `dkms`, running `%post` scriptlets that
  depend on the rpm macro system) — use the native `rpmbuild`
  / `debuild` flow.
- You need a full source-package (`.dsc` + `.orig.tar.gz`) for
  Debian upload — `nfpm` produces binary packages only.
- You want OS-image-level packaging (Flatpak, Snap, AppImage,
  OCI) — use the respective native builders; `nfpm` is for
  classic distro package managers.
