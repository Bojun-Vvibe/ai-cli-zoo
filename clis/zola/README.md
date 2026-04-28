# zola

- **Repo**: https://github.com/getzola/zola
- **Version**: v0.22.1
- **License**: EUPL-1.2 (`LICENSE`)

## What it is

A fast static site generator written in Rust. Single binary, zero runtime dependencies, with built-in Sass compilation, syntax highlighting, link checking, image processing, and a Tera templating engine.

## Why it's in the zoo

Compared to Hugo (Go) and Jekyll (Ruby), Zola targets the "drop one binary in CI and ship a site" workflow. No plugin ecosystem to babysit, everything is in-tree, and builds are typically sub-second on small-to-medium sites.

## Install

```sh
brew install zola
```

## Quick example

```sh
zola init my-site && cd my-site && zola serve
```
