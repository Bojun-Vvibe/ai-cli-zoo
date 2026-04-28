# bat-extras

- **Repo**: https://github.com/eth-p/bat-extras
- **Version**: v2024.08.24
- **License**: MIT (`LICENSE.md`)

## What it is

A bundle of shell scripts that compose `bat` (the syntax-highlighting `cat` replacement) with other Unix tools to produce highlighted variants of common workflows: `batgrep` (ripgrep with bat-rendered context), `batman` (man pages through bat), `batpipe` (pager filter for archives/binaries via `less`), `batdiff` (git/file diffs through bat or delta), `batwatch`, and `prettybat`. Each script is self-contained POSIX shell and can be installed individually.

## Why pick this over alternatives

Pick bat-extras over rolling your own aliases or installing `delta`+`ripgrep`+`less` filters separately when you want **one coherent bat-themed surface across grep, man, less, and diff** that respects your existing `bat` theme/config and ships as a single Homebrew formula instead of N independent tools to configure.

## Install

```sh
brew install bat-extras
```

## Quick example

```sh
batgrep 'TODO' src/
batman tar
batdiff HEAD~1 HEAD
```
