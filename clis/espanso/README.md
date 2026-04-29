# espanso

> **Cross-platform text-expander as a single Rust daemon** — types
> a triggered shorthand (`:email`, `:date`, `;sig`) and espanso
> instantly replaces it with the expanded snippet, optionally
> running shell / date / clipboard / form / regex transforms before
> the replacement lands. Pinned to **v2.3.0**
> ([LICENSE](https://github.com/espanso/espanso/blob/dev/LICENSE),
> GPL-3.0).

Source: <https://github.com/espanso/espanso>

## TL;DR

`espanso` watches every keystroke at the OS input layer (X11 / Wayland
on Linux, the Quartz event tap on macOS, low-level keyboard hook on
Windows), and when a configured trigger matches it deletes the
trigger and types the replacement. Snippets live in YAML files under
`~/Library/Application Support/espanso/match/` (macOS) /
`~/.config/espanso/match/` (Linux) / `%APPDATA%\espanso\match\`
(Windows); reload is automatic on file save. Triggers can be plain
strings (`:hello`), regex patterns (`:greet/(\w+)/`), or hotkey
combos. Replacements can be plain text, multi-line blocks, dynamic
date/clipboard tokens, shell command output, AppleScript / PowerShell
output, or interactive form prompts that pop a small window asking
for inputs and substitute them into a template. The Hub at
`hub.espanso.org` is a community-curated package registry — install
an `accented-letters` or `markdown-everywhere` pack with one
`espanso install` and it merges into your matchset.

## Install

```bash
# Homebrew (macOS / Linux)
brew install espanso

# macOS — register and start the daemon
espanso service register
espanso start

# Linux (Wayland) — AppImage from the release page
curl -L -o espanso.AppImage \
  https://github.com/espanso/espanso/releases/download/v2.3.0/Espanso-X11.AppImage
chmod +x espanso.AppImage && sudo mv espanso.AppImage /usr/local/bin/espanso
espanso service register && espanso start

# Windows
# winget install espanso

# verify
espanso --version    # espanso 2.3.0
espanso status       # espanso is running
```

First run drops a `base.yml` with a few example matches; edit it and
the change is live within ~1 s — no restart.

## License

GPL-3.0 — see
[LICENSE](https://github.com/espanso/espanso/blob/dev/LICENSE).
Copyleft: redistributing a modified espanso binary requires shipping
the modifications under GPL-3.0. The snippets you write are *your*
content and are not affected by espanso's license.

## One Concrete Example

```yaml
# ~/.config/espanso/match/base.yml
matches:
  # 1. plain text
  - trigger: ":sig"
    replace: |
      —
      Bojun
      bojun@example.com

  # 2. dynamic date
  - trigger: ":today"
    replace: "{{mydate}}"
    vars:
      - name: mydate
        type: date
        params: { format: "%Y-%m-%d" }

  # 3. shell command output (pipe a git short SHA into the doc you are writing)
  - trigger: ":sha"
    replace: "{{out}}"
    vars:
      - name: out
        type: shell
        params: { cmd: "git rev-parse --short HEAD" }

  # 4. interactive form — pops a window asking for {name} and {project},
  #    fills the template
  - trigger: ":pr"
    replace: |
      ## {{form1.title}}

      Closes #{{form1.issue}}

      cc @{{form1.reviewer}}
    vars:
      - name: form1
        type: form
        params:
          layout: |
            Title: [[title]]
            Issue: [[issue]]
            Reviewer: [[reviewer]]

  # 5. regex trigger — `:greet/Alice/` expands to "Hello, Alice!"
  - regex: ":greet/(?P<name>\\w+)/"
    replace: "Hello, {{name}}!"
```

## Niche It Fills

**The "expand once, work in every text field" layer.** Editor-side
snippet engines (UltiSnips, VS Code snippets, JetBrains Live
Templates) only fire inside the editor; espanso fires in *every*
focused text input — Slack, browser, mail client, JIRA web UI,
terminal, native macOS apps. For the writing tasks where you bounce
across surfaces (fill a form in a browser → reply on Slack → write a
commit message → drop a snippet in Notion), one set of triggers
works in all of them.

## Why use it

1. **One snippet definition wins everywhere.** Single YAML edited
   once propagates to every focused text field; no per-app config,
   no per-editor extension to maintain.
2. **Programmable replacements, not just static text.** Date /
   clipboard / shell-output / form / regex variables make snippets
   *generative* — `:sha` expands to the current repo's HEAD short
   SHA, `:pr` pops a form to author a PR template inline.
3. **Live reload + Hub packages.** Save the YAML, the match is
   active in <1 s; `espanso install <pkg>` adds curated snippet
   packs (accented letters, currency symbols, common emoji shortcuts,
   markdown helpers) without hand-authoring them.

For an LLM-CLI workflow, `:llm-prompt-review` can expand into a
multi-line prompt template you reuse 30 times a day across [`mods`](../mods/),
[`aichat`](../aichat/), web chat UIs, and Slack-bot prompts — without
copy-pasting from a notes file.

## Vs Already Cataloged

- **Vs editor snippet systems (out of catalog scope):** orthogonal.
  Editor snippets give you tab-stops and AST-aware insertion in the
  editor; espanso gives you the same triggers in the browser, Slack,
  mail, and the terminal. Many users run both.
- **Vs [`pet`](../pet/):** pet is a *shell command* snippet manager
  with fzf-driven recall (`pet search` → run); espanso is a
  *keystroke-level* expander that works in any text field, not just
  the shell. pet stores reusable shell one-liners; espanso stores
  reusable text — they overlap on "save a thing I retype" but
  pet executes commands, espanso types text.

## Caveats

- **GPL-3.0.** Vendoring a modified espanso binary into a closed
  product requires releasing modifications under GPL-3.0. Using
  espanso as a tool to author content does not affect that content's
  license.
- **OS input-layer permissions.** macOS requires Accessibility +
  Input Monitoring grants in System Settings → Privacy & Security
  before any expansion fires; Linux/Wayland needs the X11 backend
  AppImage on most desktops because Wayland lacks a stable global
  input-injection API.
- **Triggers fire in password fields.** Anything that types into the
  focused input will type into a password box too; pick triggers
  (`:`-prefixed, `;;`-prefixed) you would never type accidentally
  inside a credential.
- **Daemon must be running.** `espanso service register && espanso
  start` (or the Windows installer's autostart) — if the daemon is
  not running, no expansions happen and there is no UI affordance
  saying so. `espanso status` is the quick check.
- **Hub packages are user-contributed.** Read a package's `package.yml`
  before installing — it can ship arbitrary shell-output triggers
  that run on your machine.
