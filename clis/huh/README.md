# huh

> **Build interactive forms and prompts in your terminal.**
> A Go library *and* a standalone `huh` CLI from the Charm
> ecosystem that turns "ask the user a few structured questions
> from a shell script" into a single composable command — text
> inputs, single-select, multi-select, confirmations, with
> validation and a Bubble Tea–powered TUI underneath.
> Pinned to **v2.0.3**
> ([LICENSE](https://github.com/charmbracelet/huh/blob/main/LICENSE),
> MIT).

Source: <https://github.com/charmbracelet/huh>

## TL;DR

`huh` is what you reach for when a shell script needs to ask the
user a question and `read -p` is no longer enough. It ships in
two shapes:

1. **Go library** (`github.com/charmbracelet/huh`) — declare a
   `huh.NewForm(...)` with grouped fields (Input, Select,
   MultiSelect, Confirm, Note, Text), bind each to a Go variable,
   call `.Run()`, and the user gets a polished single-screen TUI
   with arrow-key navigation, `/`-search inside long lists,
   live validation, and theme-able styling.
2. **Standalone `huh` CLI** — the same widgets exposed as
   subcommands (`huh input`, `huh select`, `huh confirm`, etc.)
   that print the chosen value(s) to stdout, so a plain `bash`
   script can say `name=$(huh input --title "Your name")` and
   get a Charm-quality prompt with one line.

It is the prompt layer underneath `gum`'s newer subcommands and
the form engine inside several Charm products (`mods`, `freeze`,
`vhs` config wizards).

## Install

```bash
# Homebrew (macOS / Linux) — installs the standalone CLI
brew install charmbracelet/tap/huh

# Go (library + CLI)
go install github.com/charmbracelet/huh/cmd/huh@latest

# As a Go module dependency
go get github.com/charmbracelet/huh@v2.0.3

# Nix
nix-shell -p huh

# verify
huh --version
```

## License

MIT — see
[LICENSE](https://github.com/charmbracelet/huh/blob/main/LICENSE).
Permissive, suitable for embedding inside proprietary Go binaries
or shipping the CLI inside container images. Charm's standard
contributor terms; no CLA gating drive-by patches.

## One Concrete Example

```bash
# 1. Ask a single question from a shell script
name=$(huh input --title "What's your name?" --placeholder "Ada")
echo "Hello, $name."

# 2. Pick one item from a list
env=$(huh select --title "Deploy to" --options dev staging prod)
echo "Deploying to $env"

# 3. Pick many items (space toggles, enter confirms)
features=$(huh multiselect \
    --title "Enable features" \
    --options auth billing search analytics)
# $features is a newline-separated list

# 4. Confirm before a destructive action
huh confirm --title "Drop the production database?" \
    --affirmative "Yes, drop it" \
    --negative "Cancel" \
  || { echo "aborted"; exit 1; }

# 5. Library form (Go) — a real onboarding wizard in 20 lines
cat > main.go <<'EOF'
package main

import (
    "fmt"
    "github.com/charmbracelet/huh"
)

func main() {
    var (
        name  string
        role  string
        langs []string
        ok    bool
    )
    form := huh.NewForm(
        huh.NewGroup(
            huh.NewInput().Title("Name").Value(&name).
                Validate(func(s string) error {
                    if s == "" { return fmt.Errorf("required") }
                    return nil
                }),
            huh.NewSelect[string]().Title("Role").
                Options(huh.NewOptions("dev", "ops", "pm")...).
                Value(&role),
            huh.NewMultiSelect[string]().Title("Languages").
                Options(huh.NewOptions("go", "rust", "ts", "py")...).
                Value(&langs),
            huh.NewConfirm().Title("Looks good?").Value(&ok),
        ),
    )
    if err := form.Run(); err != nil { panic(err) }
    fmt.Printf("%s (%s) -> %v ok=%v\n", name, role, langs, ok)
}
EOF
go mod init demo && go get github.com/charmbracelet/huh@v2.0.3
go run .
```

## Niche It Fills

**The "structured prompt for shell + Go" gap.** Bash has `read`
and `select`; that's it — no validation, no multi-select, no
nice rendering, no way to compose multiple questions into one
screen. Python has `inquirer` / `questionary`; JS has `prompts` /
`enquirer`; Go and shell were both underserved. `huh` is the
Charm answer to both at once: the same widget engine drives
either a Go library call or a one-line shell invocation, with
identical visuals. For agent / CLI tooling, it's the right
choice when you need a *blocking* "ask the human three things,
validate, return" interaction that feels modern without
hand-rolling a TUI.

## Why use it

Three things `huh` does that ad-hoc prompt code doesn't:

1. **Grouped forms with one-screen layout.** Multiple fields
   render in a single Bubble Tea view; tab moves between them;
   the whole form submits atomically. You don't get the typical
   "prompt → answer → prompt → answer" scrollback noise.
2. **Validation as a first-class field.** Every Input / Text
   field takes a `Validate func(string) error`; the error
   renders inline under the field and the form refuses to
   advance until it passes. No "the script accepted my empty
   answer and then crashed".
3. **Same widgets in library and CLI form.** A Go program can
   start as a shell script using the `huh` binary, then graduate
   to a real Go binary with the *exact same* prompts — no
   rewrite of the UX layer.

For an LLM-CLI workflow, `huh` is the **human-in-the-loop input
layer**: an agent that needs to ask "which of these 5 candidate
patches do you want me to apply?" gets a real multi-select with
search instead of "type 1, 2, or 3 (comma-separated)".

## Vs Already Cataloged

- **Vs [`gum`](../gum/):** `gum` is the older, broader Charm
  shell-glue toolkit (spinners, styled output, file pickers,
  pagers, *plus* prompts). `huh` is the focused successor for
  the *form* slice — richer field types, proper grouped layout,
  validation. Use `gum` when you want one tool that also styles
  output and renders spinners; use `huh` when you specifically
  want a multi-field form and `gum`'s flat single-prompt model
  feels limiting. They coexist; Charm itself uses both.
- **Vs `dialog` / `whiptail`:** ncurses-era TUIs with grey-on-blue
  boxes and a 1990s feel. `huh` is the modern equivalent —
  same idea (block, ask, return), 30 years of UX evolution.
- **Vs writing prompts inline with `read`:** Works for one
  yes/no question. Fails the moment you need validation,
  multi-select, or "ask three related things and let the user
  go back and edit the first one".

## Caveats

- **Not a general-purpose TUI framework.** `huh` is forms and
  only forms — modal, blocking, "ask N things and exit". For a
  long-lived interactive TUI (dashboards, file managers, chat
  UIs), reach for Bubble Tea directly; `huh` is built on top
  of Bubble Tea precisely so you can mix them, but `huh` alone
  won't give you a non-modal UI.
- **CLI subcommands print to stdout, errors to stderr.** When
  capturing in `$(...)`, remember that a Ctrl-C at the prompt
  exits non-zero with empty stdout — always check `$?` after
  `$(huh ...)` if cancellation should abort the script.
- **Theme is global per form.** You can pick from `huh.ThemeBase`,
  `huh.ThemeCharm`, `huh.ThemeDracula`, `huh.ThemeBase16`,
  `huh.ThemeCatppuccin`; you cannot easily restyle individual
  fields without writing a custom `*huh.Theme`.
- **Requires a TTY.** Like every Bubble Tea program, `huh` needs
  `/dev/tty` open. Piping the output is fine; piping the *input*
  (`echo y | huh confirm`) does not work — use `--default` flags
  or a non-interactive code path for CI.
- **v2 dropped some v1 helpers.** If you find old tutorials,
  the Group/Field API was reshaped between v1 and v2; pin to
  `v2.0.3` and follow the current README.
