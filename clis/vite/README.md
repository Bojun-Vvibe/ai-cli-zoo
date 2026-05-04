# vite

## Overview

`vite` is a frontend build tool and dev server. In development it serves source
files over native ES modules with on-demand transforms, so cold start and HMR
stay fast as the project grows. For production it bundles with Rollup (and is
moving toward a Rust-based bundler, Rolldown), producing optimized static
assets.

It ships first-class support for TypeScript, JSX, CSS modules, PostCSS, and
asset imports out of the box, and has framework presets for React, Vue, Svelte,
Solid, Preact, Lit, and others. The `vite` CLI exposes three core commands:
`vite` (dev), `vite build`, and `vite preview`.

## Repo URL

https://github.com/vitejs/vite

## Version

v8.0.10 (latest in the v8 line)

## License

MIT — upstream LICENSE file: [`LICENSE`](https://github.com/vitejs/vite/blob/main/LICENSE)

## Install

Scaffold a new project (recommended starting point):

```sh
npm create vite@latest my-app
cd my-app
npm install
```

Or add Vite to an existing project:

```sh
npm install --save-dev vite
```

`pnpm`, `yarn`, and `bun` are all supported equivalently.

## Quick example

Inside a project with an `index.html` at the root:

```sh
npx vite          # start dev server with HMR on http://localhost:5173
npx vite build    # produce a production build in ./dist
npx vite preview  # serve the built ./dist locally to sanity-check
```

Minimal `vite.config.ts`:

```ts
import { defineConfig } from 'vite'

export default defineConfig({
  server: { port: 5173 },
  build:  { sourcemap: true },
})
```

## When to choose it

- You are building a single-page app, component library, or static site and
  want fast dev startup and HMR without hand-rolling a webpack config.
- You want a tool that works the same way across React, Vue, Svelte, Solid,
  and vanilla TS projects.
- You need a modern dev server with native ESM, TS, and CSS handling out of
  the box.
- You want a healthy plugin ecosystem (Rollup-compatible) for things like PWA,
  legacy browser support, image optimization, and SSR.

## When NOT to choose it

- You target environments without ES module support and cannot ship a legacy
  build (Vite assumes modern browsers by default).
- You need a full meta-framework with routing, data loading, and server
  rendering out of the box — pick a framework built on top of Vite (e.g.
  SvelteKit, Nuxt, Astro, Remix) instead of using `vite` directly.
- Your project is a pure Node.js backend with no frontend assets — use a
  TypeScript runner like `tsx` or `ts-node`, or a bundler like `tsup`.
- You are heavily invested in a webpack-only plugin you cannot replace; the
  Rollup plugin API is similar but not identical.
