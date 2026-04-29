# cloc

> **Counts blank lines, comment lines, and physical lines of source
> code across 200+ programming languages from a single, dependency-
> free Perl script — no language servers, no build, no Node/Python
> runtime, just `cloc <path>` and a grouped table by language.**
> Pinned to **v2.08**,
> [LICENSE](https://github.com/AlDanial/cloc/blob/master/LICENSE),
> GPL-2.0.

Source: <https://github.com/AlDanial/cloc>

## TL;DR

`cloc` is the boring, correct answer to "how big is this codebase,
really?" It is a single ~17 k-line Perl script (`cloc` itself, no
modules to install on any box that already has `perl`) that walks a
tree, classifies each file by extension + shebang + content
heuristics, strips comments and blank lines per the language's
actual comment grammar (not a regex over `//` and `#`), and prints
a table of `files / blank / comment / code` grouped by language.
It understands 200+ languages out of the box (Rust, Go, TypeScript,
Svelte, Nix, Terraform, Dockerfile, GraphQL, Solidity, Zig,
Carbon, …), it knows about embedded languages (HTML with inline
`<style>` and `<script>` are split into three buckets), and it
de-duplicates identical files so a vendored copy of `lodash.js`
under `node_modules/` and `vendor/` is counted once. Output modes
include human-readable tables, CSV, JSON, YAML, XML, SQL, and a
"diff" mode (`cloc --diff a/ b/`) that reports added/removed/
modified/same lines per language between two trees or two git
commits — the canonical use is `cloc --diff $(git merge-base
main HEAD) HEAD` to size a PR by language without GitHub's UI.

## Install

```bash
# Homebrew (macOS / Linux)
brew install cloc

# Debian / Ubuntu
sudo apt-get install cloc

# Arch
sudo pacman -S cloc

# any box with perl ≥ 5.10 (no CPAN modules required)
curl -L https://github.com/AlDanial/cloc/releases/download/v2.08/cloc-2.08.pl \
  -o /usr/local/bin/cloc && chmod +x /usr/local/bin/cloc

# Docker (no perl on host)
docker run --rm -v "$PWD:/tmp" aldanial/cloc .

# verify
cloc --version    # 2.08
```

## License

GPL-2.0 — see
[LICENSE](https://github.com/AlDanial/cloc/blob/master/LICENSE).
The script is freely redistributable; modifications to the script
itself must remain GPL-2.0. Counting GPL'd code with `cloc` does
not infect the counted code with GPL — `cloc` reads files, it
does not link against them.

## One Concrete Example

```bash
# 1. size a repo by language, exclude vendored + generated dirs
cloc . \
  --exclude-dir=node_modules,vendor,dist,build,.next,target \
  --exclude-ext=lock,min.js,min.css

# 2. size a PR by language vs main (no GitHub UI needed)
cloc --diff $(git merge-base main HEAD) HEAD --exclude-dir=node_modules

# 3. machine-readable JSON for a CI quality gate
#    (fail PR if it adds > 500 lines of code in a single commit)
added=$(cloc --diff $(git merge-base main HEAD) HEAD \
          --exclude-dir=node_modules --json \
          | jq '[.[] | select(.code) | .code.added] | add // 0')
[ "$added" -gt 500 ] && {
  echo "PR adds $added lines — split it"; exit 1;
}

# 4. compare two release tags by language
cloc --diff v1.0.0 v2.0.0 --git --exclude-dir=node_modules

# 5. one-shot SQL dump for a multi-repo dashboard
cloc --sql=- --sql-project=myrepo . | sqlite3 sizes.db
```

## Why This, Not Another

1. **Comment-aware, not regex-aware.** A `//` inside a Rust string
   literal, a `#` inside a Bash heredoc, a `/* */` inside a TypeScript
   template literal — `cloc` strips them correctly because it knows
   each language's lexical rules, not because it greps for comment
   markers. The line counts you get out are the line counts a human
   reviewer would arrive at.
2. **Zero install on any Unix.** It is *one* Perl script; `perl`
   ships on macOS, every Linux distro, every Docker base image,
   every CI runner. There is no "but does it work on the build
   container" failure mode. Compare against `tokei` (Rust, fast,
   needs a binary download) or `scc` (Go, ditto) — both are
   excellent but require a build artifact per arch.
3. **`--diff` is the killer feature.** Most competitors count.
   `cloc --diff` *compares* two trees / commits / tags and emits
   added / removed / modified / same per language. Wire that into
   CI to size PRs, into a release script to size each release,
   into `git bisect` to find the commit that doubled the JS
   surface area.

For an LLM-CLI workflow, `cloc --json . | jq` is the cheapest way
to give a model a structured "this repo is 14k lines TypeScript,
3k lines Python, 800 lines SQL" preamble before asking it to plan
a refactor — far smaller than feeding a `tree` listing or a
[`files-to-prompt`](../files-to-prompt/) dump.

## Vs Already Cataloged

- **Vs [`tokei`](../tokei/):** `tokei` is the Rust answer to the
  same problem — much faster on huge trees (parallel, mmap), nicer
  default output. Pick `tokei` for raw speed on monorepos and
  for embedding via its Rust crate; pick `cloc` when you cannot
  install a binary on the target box (CI runner without sudo,
  ancient bastion host) or when you need `--diff` between commits,
  which `tokei` does not have.
- **Vs [`scc`](../scc/):** `scc` is the Go answer — also faster
  than `cloc`, also computes COCOMO estimates and complexity
  proxies. Pick `scc` when you want the cost / complexity numbers
  in the same pass; pick `cloc` when you want correctness on
  obscure languages (`scc` has a smaller language matrix) and
  the `--diff` mode.
- **Vs [`scc`](../scc/) + [`tokei`](../tokei/) together:** in
  practice teams pick *one* of the three. `cloc` is the safe
  default for "I need a number in the next 30 seconds on a box I
  don't control."

## Caveats

- **Perl, not a static binary.** Fast enough for almost everything
  (it counts the Linux kernel in ~30 s), but on multi-million-line
  monorepos `tokei` / `scc` are 10–50× faster because they are
  parallel and native. On a 100k-line app you will not notice.
- **GPL-2.0, not MIT/Apache.** Redistributing the script (e.g.
  vendoring it into your repo) is fine; modifying the script and
  shipping the modified version requires you to honour GPL-2.0
  obligations on the modified `cloc` portion. The output of `cloc`
  is data, not a derivative work — your codebase is not infected.
- **Heuristic language detection has edge cases.** A `.h` file is
  ambiguous between C and C++; a `.pl` file is ambiguous between
  Perl and Prolog. `cloc` guesses well but ships `--read-lang-def`
  / `--force-lang` overrides for the cases where it guesses wrong.
- **`--diff` requires both trees on disk** (or `--git` mode that
  checks them out for you). It is not a streaming diff over a
  patch file; for very large repos plan for the disk + time cost
  of two checkouts.
- **No "complexity" or "cost" output.** If you want cyclomatic
  complexity, COCOMO dollar estimates, or a maintainability index,
  `scc` is the tool. `cloc` is deliberately scoped to *counting*.
