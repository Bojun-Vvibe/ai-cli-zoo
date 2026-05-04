# zx

## Overview

`zx` is a small wrapper around Node.js that makes shell scripting in JavaScript
or TypeScript pleasant. It exposes a `$` template tag that runs commands in a
subshell and returns a promise resolving to the process result, plus helpers
for `cd`, `fetch`, `glob`, `question`, `chalk`, `fs`, `os`, and friends — so a
script doesn't have to import a dozen utility libraries to do basic things.

It is aimed at the niche between "ad-hoc bash one-liner" and "full Node CLI
project": automation scripts, CI glue, release tooling, repo chores. Scripts
can be a `.mjs`, `.ts`, `.md` (executes fenced code blocks), or even a remote
URL.

## Repo URL

https://github.com/google/zx

## Version

8.8.5 (released 2025-10-19)

## License

Apache-2.0 — upstream LICENSE file: [`LICENSE`](https://github.com/google/zx/blob/main/LICENSE)

## Install

Requires Node.js 12.17.0+.

```sh
npm install -g zx
```

Or run a one-off without installing:

```sh
npx zx script.mjs
```

A standalone single-file binary build is also published with releases.

## Quick example

`deploy.mjs`:

```js
#!/usr/bin/env zx

await $`git pull --rebase`
const branch = (await $`git rev-parse --abbrev-ref HEAD`).stdout.trim()

if (branch !== 'main') {
  console.log(chalk.yellow(`refusing to deploy from ${branch}`))
  process.exit(1)
}

await $`npm ci`
await $`npm run build`

const tag = `release-${Date.now()}`
await $`git tag ${tag}`
await $`git push origin ${tag}`

console.log(chalk.green(`shipped ${tag}`))
```

Run it:

```sh
chmod +x deploy.mjs
./deploy.mjs
# or
zx deploy.mjs
```

## When to choose it

- You want shell-style command piping but with real variables, async/await,
  arrays, JSON, and error handling.
- Your team already speaks JavaScript/TypeScript and bash scripts have grown
  past the comfort zone (functions, error paths, structured config).
- You want the same script to run on Linux, macOS, and Windows without
  rewriting it for PowerShell.
- You want safe argument interpolation by default (the `$` tag escapes values
  rather than splatting them into a shell string).

## When NOT to choose it

- You cannot install Node.js on the target machine, or you need scripts that
  run in environments where only POSIX `sh` is guaranteed.
- The script is genuinely two lines of `cp` and `mv` — plain bash is fine and
  has no startup cost.
- You need a long-running daemon or a published CLI tool with subcommands,
  flags, and help text — reach for a real CLI framework instead.
- Strict supply-chain policies forbid pulling Node packages for system
  automation.
