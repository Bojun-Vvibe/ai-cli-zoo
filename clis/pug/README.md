# pug

> **Terminal UI for Terraform / OpenTofu** that drives
> `init` / `plan` / `apply` / `destroy` across many
> modules and workspaces in parallel from one keyboard-
> driven dashboard. Pinned to **v0.6.5**
> ([LICENSE](https://github.com/leg100/pug/blob/master/LICENSE),
> MPL-2.0).

Source: <https://github.com/leg100/pug>

## TL;DR

`pug` walks a directory tree, auto-detects every
Terraform / OpenTofu module (any directory containing
`*.tf`), discovers each module's workspaces, and
renders them as a `bubbletea` TUI: a sortable table of
modules on the left, the workspaces under each module
collapsible inline, and a tabbed details pane on the
right showing the run history, current state, and the
live log of whatever command is executing on the
highlighted row. Single keys queue work — `i` `init`,
`v` `validate`, `f` `format`, `p` `plan`, `a` `apply`,
`d` `destroy`, `s` open a state browser — and
multi-select (`Space` to mark, `Ctrl-a` to mark all
filtered rows) fans the same verb across every selected
module / workspace, so "init the 14 newly cloned
modules", "plan everything matching `^prod-`", or "show
me the diff between every staging and prod workspace"
collapse from a shell loop into one keystroke. Plans
land as artifacts you can re-`apply` later from the run
history without re-planning; the dependency graph is
honoured so `terragrunt`-style cross-module deps run in
the right order; and the whole thing shells out to your
local `terraform` / `tofu` binary so plugin caches,
backends, and provider auth keep working unchanged.

## When to use

- You operate **many small Terraform/OpenTofu modules**
  (per-account, per-region, per-service) and the daily
  loop is "init + plan across N of them" rather than
  "edit one big module".
- You want a **persistent run history with replayable
  plan artifacts** without standing up Atlantis,
  Spacelift, or Terraform Cloud.
- You want to **multi-select and fan a verb** (`plan`,
  `apply`) across a filtered set without writing a
  shell loop or `for d in modules/*; do ...`.

## When NOT to use

- You manage **one monolithic root module** — vanilla
  `terraform plan` / `apply` is simpler; pug's value
  comes from cross-module orchestration.
- You need **policy gates, approvals, drift detection,
  or audit trails** for a team — that is the niche of
  Atlantis / Spacelift / env0 / Terraform Cloud, not a
  local TUI.
- You want **headless CI execution** — pug is
  interactive-first; CI should call `terraform` /
  `tofu` directly (or use a CI-shaped runner like
  `terragrunt run-all`).

## Install

```bash
# Homebrew (tap)
brew install leg100/tap/pug

# Go install
go install github.com/leg100/pug@latest

# prebuilt binaries
# https://github.com/leg100/pug/releases

# verify
pug --version    # pug version v0.6.5
```

## Basic usage

```bash
# from the root of a repo with many TF/OpenTofu modules
pug

# point at a specific tree
pug -w ./infra

# use OpenTofu instead of terraform
pug --program tofu

# pre-set max concurrent runs (default 3)
pug --max-tasks 8
```

Inside the TUI: `?` opens the keymap, `Tab` cycles
panes, `/` filters, `Space` multi-selects, then
`p` / `a` / `d` queue plan / apply / destroy across the
selection. Plans live in the run history (`R` tab) and
can be applied later with `a` from there.

## Pairs with

- [`opentofu`](../opentofu/) / `terraform` — pug shells
  out to one of these; pin via `--program`.
- [`atlas`](../atlas/) / [`pulumi`](../pulumi/) —
  orthogonal IaC tools; pug is Terraform/OpenTofu only.
- [`terragrunt`](https://terragrunt.gruntwork.io/) —
  pug honours module dependencies discovered from
  `terragrunt.hcl` for ordered fan-out.
- [`infracost`](../infracost/) — cost diff on the
  plan artifacts pug stores in its run history.
