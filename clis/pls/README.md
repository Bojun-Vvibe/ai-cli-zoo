# pls

- **Repo:** https://github.com/pls-rs/pls
- **Version:** v0.0.1-beta.9 (latest tag, Rust rewrite under active development)
- **License:** GPL-3.0 ([LICENSE](https://github.com/pls-rs/pls/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `cargo install pls` · prebuilt binaries on the GitHub releases page · `brew install pls` (community tap)

## What it does

`pls` is a `prettier`-than-`ls` directory lister written in
Rust that treats the listing as a *configurable view* over the
filesystem instead of a fixed columnar dump. Per-directory
`.pls.yml` files declare which entries to surface, hide, group,
or annotate — so a repo can ship "when someone runs `pls` in
this folder, show `src/` and `tests/` first, hide
`node_modules/` / `target/` / `__pycache__/` even though they
exist, and tag `Cargo.lock` with a 'lockfile' badge". On the
display side it reuses the modern-`ls` toolkit (Nerd Font icons
keyed to MIME / extension / filename, true-colour syntax-style
highlighting, git status badges next to each entry, octal mode
and human-readable sizes, owner / group resolution) but adds
`--details` columns that are themselves configurable per
directory: a docs folder can render `created`, `modified`,
`words`, `headings`; a code folder can render `loc`, `lang`,
`tests-passing` from a hook. Filter / sort / group are
expression-driven (`--filter "name ~ '\\.rs$'"`, `--sort
modified-desc`, `--group ext`) rather than the loose alphabet
soup of `ls -ltrSh`, and every flag has a long form so scripted
invocations stay readable. Two distinct projects share the
name "pls" — the Python wrapper at `dhruvkb/pls` is the
original; `pls-rs/pls` is the Rust rewrite this entry
documents (faster startup, single static binary, no Python
runtime).

## What's interesting

- **Per-directory `.pls.yml`** — the *project* declares the
  default listing view, so newcomers see the curated layout
  (`src/` first, build artefacts hidden) without knowing the
  flag incantation.
- **Configurable detail columns** — beyond size / mtime /
  perms, you can wire columns like `loc` / `tests` / `words`
  via hooks; the lister becomes a project-aware dashboard.
- **Real expression filter / sort / group** — `--filter "name
  ~ '\\.test\\.ts$' and size > 1k"` instead of `ls | grep | xargs`.
- **Git-aware out of the box** — staged / modified / untracked
  / ignored badges next to each entry without a separate
  `git status` lookup or an `ls`-replacement-plus-zsh-plugin
  combo.
- **Plays well with [`eza`](../eza/) / [`lsd`](../lsd/)** —
  pls leans on per-directory config and project-specific
  views; eza/lsd lean on universal `ls`-replacement defaults.
  Use pls inside a repo that benefits from a curated entry
  view; use eza/lsd as your global `ls` alias.

## AI-native angle

When an LLM agent calls `ls` to orient itself in an unfamiliar
repo, it gets back a noisy alphabet-soup dump (`.git/`,
`.venv/`, `node_modules/`, `target/`, `dist/`, `coverage/`,
seven config dotfiles, then the three directories that
actually matter) and burns context tokens on the noise. A
project that ships a `.pls.yml` declaring "for `pls` callers,
show `src/`, `tests/`, `docs/`, `README.md`, `pyproject.toml`
in that order and hide everything else by default" turns the
same orientation call into a curated, tokens-cheap summary —
the human and the agent see the same authored "table of
contents" instead of the raw filesystem. The configurable
detail columns extend this to richer signals (LOC per dir,
last-commit author, test pass/fail) so an agent can read repo
shape from one tool call instead of three. Pairs with
[`onefetch`](../onefetch/) (one-shot repo summary card),
[`tokei`](../tokei/) (LOC counts that pls can surface as a
column), and [`scc`](../scc/) (cocomo + complexity columns)
when the agent's first move in a new repo is "what is this
project and where do I start reading?".