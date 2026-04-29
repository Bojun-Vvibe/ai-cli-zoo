# license

> **Single-binary command-line license-text generator: prints any
> SPDX license (MIT, Apache-2.0, GPL-3.0, MPL-2.0, ISC, BSD variants,
> Unlicense, …) to stdout or `LICENSE`, with author / year filled in
> from CLI flags or `git config`.** Pinned to **v5.0.4**, MIT
> ([LICENSE](https://github.com/nishanths/license/blob/master/LICENSE)).

- **Repo:** https://github.com/nishanths/license
- **Latest version:** v5.0.4
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `dev-experience` / `project-bootstrap`
- **Language:** Go
- **Install:** `go install github.com/nishanths/license/v5@latest` or
  `brew install license`

## What it does

`license` is a zero-dependency Go binary that ships an embedded copy
of the canonical SPDX license corpus and writes the chosen license
text to stdout (default) or to a file with `-o LICENSE`. Pick by
short SPDX id — `license mit`, `license apache-2.0`,
`license gpl-3.0`, `license mpl-2.0`, `license bsd-3-clause`,
`license unlicense`, `license cc0-1.0` — and the templated
placeholders (`<year>`, `<copyright holders>`) are filled from
`-n "Jane Doe"` / `-y 2026` flags or, when omitted, from
`git config user.name` and the current year. `license ls` lists every
SPDX id known to the binary; `license -h <id>` prints the SPDX
metadata header (full name, OSI status, FSF status, URL) without
the body. Because the corpus is embedded at build time, the tool
works fully offline — useful in air-gapped CI and devcontainer
bootstrap.

## Why included

Every new repo needs a `LICENSE` file before the first push, and the
default options are unattractive: copy-paste from a previous repo
(stale year, wrong name), `gh repo create --license` (network round
trip, only at creation time), or hand-edit a template (typos in the
"all rights reserved" line invalidate the grant). `license` collapses
this to one offline command — `license -o LICENSE -n "Jane Doe" mit`
— that picks up author/year from git and writes a byte-identical
copy of the SPDX canonical text. For an LLM-CLI workflow that
scaffolds many small repos (one per experiment, one per scratch
mission), this removes a recurring "agent picks the wrong license
text and pollutes the diff" failure mode and gives the human a
single audited source for license text across the whole portfolio.
