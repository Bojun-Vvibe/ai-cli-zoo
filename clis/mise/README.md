# mise

- **Upstream:** https://github.com/jdx/mise
- **Version:** v2026.4.24 (2026-04-27)
- **License:** MIT — https://github.com/jdx/mise/blob/main/LICENSE (SPDX: `MIT`)

## What it does

`mise` (formerly `rtx`) is a polyglot dev-tool version manager, environment
variable manager, and task runner in one binary. It replaces tools like `nvm`,
`pyenv`, `rbenv`, `direnv`, and a lightweight `make`, reading a single
`mise.toml` per project to pin tool versions, export per-directory env vars,
and define runnable tasks. It backs onto the `asdf` plugin ecosystem plus a
native Rust core for speed.

## Example

```sh
# Pin tool versions for the current project
mise use node@22 python@3.12 go@1.23

# Run a project task defined in mise.toml
mise run test

# Show what versions are active here
mise current
```
