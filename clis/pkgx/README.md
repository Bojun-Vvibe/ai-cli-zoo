# pkgx

## What it does
Package runner and on-demand package fetcher: `pkgx <tool>@<version> <args>` resolves the tool from the pkgx pantry (a curated graph of 1000+ open-source packages with declared deps), downloads the prebuilt sandboxed binaries for your OS / arch into `~/.pkgx/`, wires up `PATH` / `LD_LIBRARY_PATH` / `DYLD_FALLBACK_LIBRARY_PATH` for that one invocation, runs the command, and exits without touching `/usr/local`, `/opt/homebrew`, or `~/.local/bin`. `pkgx +node@22 +python@3.12 -- bash` drops you into a shell where exactly those versions are first on PATH for the duration of that shell, with no shim, no shell hook, no `eval "$(... init)"`.

## Why it's interesting
Different shape from `brew` / `apt` / `nix profile install` (mutates a system-wide store) and from `mise` / `asdf` (manages versions for a project but still needs `mise install` ahead of time and a shell hook to inject shims): `pkgx` is **invocation-scoped**, so a CI job that needs `gh@2.55` for one step gets exactly that without polluting the runner image, and a developer trying out a tool runs it once with `pkgx <tool>` and the binary is cached but never `which`-able from a normal shell. The pantry is a real dep graph (each package declares its build + runtime deps in YAML), so `pkgx +ffmpeg` pulls the right `libx264` / `libvpx` versions automatically, with no system mutation.

## Niche category
Package runner — invocation-scoped on-demand binary fetcher

## Repo
https://github.com/pkgxdev/pkgx

## Version pinned
`v2.10.3`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE.txt`

## Install
```sh
brew install pkgxdev/made/pkgx
# or
curl -fsS https://pkgx.sh | sh
# or download pkgx-2.10.3+darwin+aarch64.tar.xz from the v2.10.3 release
```

## Usage examples
```sh
# One-shot run of a specific version, no install
pkgx node@22 -e 'console.log(process.version)'

# Drop into a sub-shell with multiple tool versions on PATH
pkgx +node@22 +python@3.12 +deno -- bash

# Run an arbitrary tool from the pantry without ever installing it
pkgx gh@2.55 pr list

# Cached binaries live under ~/.pkgx and are fully removable
ls ~/.pkgx
rm -rf ~/.pkgx   # uninstall everything pkgx has ever fetched
```
