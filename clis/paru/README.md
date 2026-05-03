# paru

> **A feature-rich Rust-implemented AUR helper for Arch Linux**
> that wraps `pacman` with first-class support for the Arch User
> Repository — `paru` resolves AUR + repo dependencies in one
> graph, fetches PKGBUILDs into a per-package clone tree, prompts
> the user to *review the build script and `.install` hooks* with
> a configurable diff viewer (`bat`, `vim`, `delta`) before any
> `makepkg` run, builds in clean chroots when configured, and
> hands the resulting packages back to `pacman -U` for install so
> the system package database stays canonical. Pinned to **v2.1.0**
> ([LICENSE](https://github.com/Morganamilo/paru/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/Morganamilo/paru>

## TL;DR

`paru -S firefox neovim spotify-launcher` resolves all three
across `core` / `extra` / AUR, prompts to review every PKGBUILD,
builds the AUR ones, and installs everything with one `sudo`
prompt. `paru` (no args) becomes the daily upgrade command:
`-Syu` for repos plus AUR rebuilds for any out-of-date AUR
package, with VCS-package update detection (`-Sua`).

## Install

```bash
# AUR bootstrap (needs base-devel and git first)
sudo pacman -S --needed base-devel git
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si

# Or, faster, install the binary AUR package
git clone https://aur.archlinux.org/paru-bin.git
cd paru-bin && makepkg -si

# verify
paru --version    # 2.1.0
```

Configuration lives at `/etc/paru.conf` (system) and
`~/.config/paru/paru.conf` (user). The defaults are sensible;
the most-toggled options are `BottomUp` (newest results last,
matches `yay`'s order), `CleanAfter`, and `Devel` (auto-rebuild
`-git` packages on upgrade).

## License

GPL-3.0 — see
[LICENSE](https://github.com/Morganamilo/paru/blob/master/LICENSE).
Strong copyleft; binary distribution is fine, derivative source
must remain GPL-compatible.

## One Concrete Example

```bash
# 1. install AUR + repo packages in one resolved graph
paru -S spotify visual-studio-code-bin discord_arch_electron

# 2. system upgrade including AUR rebuilds
paru -Syu

# 3. upgrade only AUR (skip pacman repos)
paru -Sua

# 4. search AUR + repos with one query, sorted by popularity
paru zoom

# 5. show full info (votes, last update, dependencies, conflicts)
paru -Si yay

# 6. review-only mode (inspect PKGBUILD, do not build)
paru --print spotify

# 7. clean orphaned dependencies (repo + AUR)
paru -c

# 8. download just the PKGBUILD source tree (offline review / packaging fork)
paru -G aur-package-name
```

## Niche It Fills

**The de-facto AUR helper on Arch Linux** for users who want
`pacman`-equivalent UX for the AUR plus the safety of mandatory
PKGBUILD review before any unknown shell script runs as your
build user. The maintained successor in the same lineage as
`yay` (and the slot most rolling-release Arch users now occupy).

## Why use it

1. **One unified resolver across repos and AUR.** No "install
   the repo deps with pacman first, then the AUR ones with the
   helper" two-step. `paru -S` figures out the full graph and
   builds bottom-up.
2. **PKGBUILD review is a first-class step, not a flag you
   forget to set.** Default workflow opens the diff between the
   currently-installed PKGBUILD and the upstream one in your
   configured pager so you actually see what changed before
   `makepkg` runs as your user. This is the single most
   important security property an AUR helper can offer.
3. **Clean-chroot builds available out of the box.** `paru
   --chroot -S <pkg>` builds inside a `devtools` chroot so the
   build environment is reproducible and host packages do not
   leak into the result — the same isolation Arch maintainers
   use for official packages.
4. **`pacman` wrapper semantics.** Every `pacman` flag works
   (`-Qs`, `-Qi`, `-R`, `-F`, `-T`, etc.) so muscle memory
   transfers — paru only intercepts the flags where AUR-aware
   behaviour matters.

## Vs Already Cataloged

- **Vs [`pacstall`](../pacstall/) / Debian-side AUR-likes:**
  pacstall is the AUR-shaped helper for Debian / Ubuntu;
  different distro, same pattern. `paru` is the Arch-native
  point in this design space.
- **Vs [`topgrade`](../topgrade/):** topgrade is the
  cross-tool, cross-package-manager system-wide upgrade
  orchestrator (run `paru -Syu` plus `brew upgrade` plus
  `cargo install-update -a` plus `flatpak update` plus
  `rustup update` in one command). Use topgrade *to call* paru
  on a multi-tool laptop; use paru directly when Arch is the
  only package source.
- **Vs `yay`:** historically the dominant AUR helper. paru is
  written in Rust by a former yay maintainer with a cleaner
  internal architecture, faster operation on large dep graphs,
  and active development; yay is in maintenance mode. The CLI
  surface is intentionally close to `yay`'s for migration.
- **Vs `aurutils` / `aura`:** aurutils is the
  small-composable-tools approach (separate `aur sync` / `aur
  build` / etc. that you wire together yourself); aura is the
  Haskell-implemented alternative with stricter defaults.
  paru sits in the "monolithic helper, sensible defaults,
  daily-driver" middle.

## Caveats

- **Arch Linux only.** Manjaro / EndeavourOS / Garuda inherit
  it; non-Arch derivatives (NixOS, openSUSE, Fedora) do not.
- **AUR is a trust hazard.** The PKGBUILD-review prompt is the
  defence; do not press `:q` on autopilot. Pinned-to-tag
  packages are safer than `-git` rolling ones, which build
  from the upstream `master` at install time.
- **`-git` packages need `Devel = true`** in `paru.conf` to
  detect upstream changes during `-Syu`; without it they look
  "up to date" forever based on the cached PKGBUILD `pkgver()`.
- **Builds happen as your user by default,** which means a
  hostile PKGBUILD has access to your home directory during
  build. For untrusted packages prefer `--chroot` (requires
  `devtools` and root for chroot setup), or build on a
  throwaway VM.
- **Not a replacement for pacman.** `paru -Syu` invokes pacman
  underneath and still requires the user to maintain
  `/etc/pacman.conf`, mirrorlist hygiene (`reflector`), and
  keyring updates — paru does not abstract those away.
