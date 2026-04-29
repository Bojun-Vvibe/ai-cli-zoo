# kubectx

- **Repo:** https://github.com/ahmetb/kubectx
- **Version:** v0.9.5 (current stable, 2026)
- **License:** Apache-2.0 ([LICENSE](https://github.com/ahmetb/kubectx/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install kubectx` (ships both `kubectx` and `kubens`) · `pacman -S kubectx` · `apt install kubectx` · prebuilt binaries on the GitHub releases page · or drop the bash script from the repo onto `$PATH`

## What it does

`kubectx` is a tiny **Kubernetes context switcher** that does
exactly one thing well: it edits the `current-context` field of
your `~/.kube/config` so the next `kubectl` invocation hits the
cluster you actually meant. Run `kubectx` with no args to list
all contexts (current one highlighted), `kubectx <name>` to switch
to one, `kubectx -` to flip back to the previous context, and
`kubectx <new>=<old>` to rename a context to something
human-readable like `prod-eu` instead of the auto-generated
`gke_my-org_europe-west1_prod`. The companion `kubens` does the
same dance for the active namespace inside the current context.

The reason it caught on is that the obvious alternative —
`kubectl config use-context …` — requires you to remember the
full context name and offers no completion or fuzzy matching.
`kubectx` exposes a one-word command, integrates with `fzf` for
interactive picking when no argument is given (highly recommended:
`brew install fzf` first), and includes shell-completion scripts
for bash/zsh/fish so tab-completion just works. Because it only
mutates `~/.kube/config`, it composes cleanly with anything else
that reads kubeconfig — Lens, k9s, helm, terraform — without any
shell wrapper or environment-variable trickery.

## Example

```bash
# list contexts; pick interactively if fzf is installed
kubectx

# switch to the staging cluster, then jump to the 'payments' namespace
kubectx staging
kubens payments

# flip back to the previous context
kubectx -
```

## When to pick it / when not to

If you touch more than one cluster or namespace in a normal day,
install it — the time-to-value is about thirty seconds and you
will use it dozens of times per session. If you live entirely in
one cluster and one namespace, you do not need it. For a richer
TUI that also browses pods/logs/events, reach for `k9s`; `kubectx`
is intentionally just the context-switching primitive, designed
to be cheap to call from prompts (`PS1`) and shell scripts.
