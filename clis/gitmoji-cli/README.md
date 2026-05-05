# gitmoji-cli

> **Interactive emoji-prefixed commit message helper that
> picks an intent emoji from the
> [gitmoji.dev](https://gitmoji.dev) catalog, then assembles
> the rest of the Conventional-Commits-shaped subject line**
> — fuzzy-search the 70+ canonical gitmoji codes
> (`:sparkles:` for a feature, `:bug:` for a fix,
> `:recycle:` for a refactor, …), pick one with arrow keys,
> answer prompts for scope / title / body, and the tool
> calls `git commit` for you. Optional `--hook` mode installs
> a `prepare-commit-msg` hook so `git commit` always opens
> the picker. Pinned to **v9.7.0** (released 2026-05-16,
> [LICENSE.md](https://github.com/carloscuesta/gitmoji-cli/blob/v9.7.0/LICENSE.md),
> MIT).

Source: <https://github.com/carloscuesta/gitmoji-cli>

## TL;DR

Two camps fight over the first character of every commit
subject line. Camp A wants Conventional Commits
(`feat:` / `fix:` / `refactor:` / `chore:`) so changelogs
generate cleanly. Camp B wants gitmoji (`✨` / `🐛` /
`♻️` / `🔧`) so the repo's `git log --oneline` reads at a
glance and tools like Sourcetree / GitHub render an emoji
column. `gitmoji-cli` is the canonical implementation for
camp B: it owns a curated 70+ emoji catalog with stable
short codes, ships an inquirer-style picker with fuzzy
search (powered by `fuse.js`), and integrates as either a
manual command (`gitmoji -c`) or a `prepare-commit-msg`
hook so `git commit` always pre-fills the right glyph.
Companion fork [`czg`](https://github.com/Zhengqbbb/cz-git)
combines both camps; pure gitmoji users stick with this one.

## Install

```bash
# Recommended: npm (or yarn / pnpm)
npm install -g gitmoji-cli@9.7.0

# macOS (Homebrew)
brew install gitmoji

# Verify
gitmoji --version          # 9.7.0
gitmoji --list             # show the full emoji catalog
```

## License

MIT — see
[LICENSE.md](https://github.com/carloscuesta/gitmoji-cli/blob/v9.7.0/LICENSE.md).
Permissive: ship inside corporate dotfiles repos and
team-wide commit-hook templates with no copyleft surface.

## Common invocations

```bash
# One-off: stage your changes, then run the picker
git add -p
gitmoji -c                # interactive: pick emoji → scope → title → body

# Install once, then every `git commit` opens the picker
gitmoji --init            # writes .git/hooks/prepare-commit-msg
git commit                # picker fires, emoji prepended to message

# Remove the hook
gitmoji --remove

# Search the catalog from the shell (handy in scripts)
gitmoji --search "test"   # matches :test_tube:, :white_check_mark:, etc.

# Update the bundled emoji catalog from gitmoji.dev
gitmoji --update

# Switch from emoji glyphs (✨) to short codes (:sparkles:) in commits
gitmoji --config          # interactive config: emojiFormat, signedCommit, scopes…

# Predefined scopes (since v9.6.0) — config-driven autocomplete
# .gitmojirc.json:
# { "scopes": ["api", "ui", "infra", "docs"] }
gitmoji -c                # scope step now shows the predefined list
```

## Why use it

- **Curated, stable catalog.** 70+ emoji with frozen short
  codes (`:sparkles:`, `:bug:`, `:recycle:`, `:lock:`,
  `:zap:`, `:fire:`, `:wrench:`, `:rocket:`, …) maintained
  alongside [gitmoji.dev](https://gitmoji.dev). Teams adopt
  the convention and `git log` reads consistently across
  contributors.
- **`prepare-commit-msg` hook integration.** `gitmoji
  --init` writes the hook for you; from then on
  `git commit` (no flags) opens the picker. Drops the
  "I forgot to use the tool" failure mode.
- **Two output formats.** `emojiFormat=emoji` ships
  `✨ feat: add X`; `emojiFormat=code` ships
  `:sparkles: feat: add X` (preferred when downstream
  changelog generators expect ASCII). Switch with
  `gitmoji --config`.
- **Predefined scopes (v9.6.0+).** A `.gitmojirc.json` at
  the repo root with `{"scopes": [...]}` turns the scope
  prompt into an autocomplete list — keeps a team's
  scopes consistent without writing a custom hook.
- **Ancestor config search (v9.7.0).** Config lookup walks
  up the directory tree, so monorepo subpackage commits
  pick up the root `.gitmojirc.json` without symlinks.
- **No LLM dependency.** Pure interactive picker — works
  offline, in air-gapped CI, on flights, with no API key
  to leak. Different category from
  [`opencommit`](../opencommit/) / [`aicommits`](../aicommits/)
  / [`gptcommit`](../gptcommit/), which generate the
  *content* from a model and the diff.

## Vs Already Cataloged

- **Vs [`commitizen`](../commitizen/) / `cz`:** `commitizen`
  is the spiritual ancestor — same picker shape,
  Conventional-Commits-first. `gitmoji-cli` ships the
  emoji catalog as the primary axis and Conventional
  Commits as a secondary scope/type prompt; `commitizen`
  is the inverse. Pick by which convention dominates your
  repo.
- **Vs [`opencommit`](../opencommit/) /
  [`aicommits`](../aicommits/) / [`gptcommit`](../gptcommit/):**
  those generate the *message text* from the staged diff
  using an LLM. `gitmoji-cli` only handles the prefix and
  the structured prompts — you write the body. Stack them:
  use `gitmoji --hook` for the prefix and pipe the diff
  through an AI tool for the body. Different concerns,
  composable.
- **Vs [`gitnr`](../gitnr/):** `gitnr` is a `.gitignore` /
  template fetcher; nothing to do with commit messages.
  Names rhyme, scopes don't overlap.
- **Vs [`cocogitto`](../cocogitto/):** `cog` is a
  Conventional-Commits validator + changelog generator +
  release-tagger. Different layer: `gitmoji-cli` writes
  the commit, `cocogitto` validates the log and cuts the
  release. They cohabit fine if your team uses gitmoji
  *as* the Conventional-Commits type prefix.
- **Vs [`lazygit`](../lazygit/) / [`gitu`](../gitu/) /
  [`gitui`](../gitui/):** those are full-screen TUIs that
  drive `git` end-to-end. `gitmoji-cli` is a one-step
  picker invoked from a normal shell. Use a TUI for
  staging-and-history work; reach for `gitmoji` at the
  commit step (or install the hook so the TUI's
  `git commit` invocation triggers it).

## Caveats

- **Node.js runtime required.** ESM Node 18+; on a server
  without Node, `brew install gitmoji` is the lighter
  install. Cold-start latency is ~200 ms (acceptable for
  an interactive picker, noticeable in tight scripts).
- **Hook collisions.** `gitmoji --init` writes
  `.git/hooks/prepare-commit-msg`. If you already use
  [`husky`](https://github.com/typicode/husky) or
  [`lefthook`](../lefthook/) and a `prepare-commit-msg`
  hook is configured there, the two will fight. Resolve
  by registering `gitmoji-cli` *inside* your hook
  manager (`husky add .husky/prepare-commit-msg "gitmoji
  --hook"`) and skipping `gitmoji --init`.
- **Catalog drift.** Run `gitmoji --update` periodically;
  the bundled catalog snapshot ages. `--update` reaches
  out to `gitmoji.dev` over HTTPS, so air-gapped envs
  must vendor the JSON manually.
- **Not a Conventional-Commits validator.** It writes the
  message; it doesn't enforce shape on incoming commits.
  Pair with [`commitizen`](../commitizen/) /
  [`cocogitto`](../cocogitto/) / `commitlint` if you need
  CI gating.
- **Maintenance cadence is mostly dependabot.** The
  v9.x line is feature-stable; recent releases are mostly
  dependency bumps with the occasional small feature
  (predefined scopes in 9.6.0, ancestor config search
  in 9.7.0). Reliable, not flashy.
