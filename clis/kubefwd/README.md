# kubefwd

- **Repo:** https://github.com/txn2/kubefwd
- **Version:** v1.25.14 (latest stable, released 2026-04-22)
- **License:** Apache-2.0 ([LICENSE](https://github.com/txn2/kubefwd/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install txn2/tap/kubefwd` · `go install github.com/txn2/kubefwd@latest` · prebuilt binaries on the GitHub release page · binary name is `kubefwd`

## What it does

`kubefwd` is **bulk port-forward + DNS bridging** for Kubernetes.
Where `kubectl port-forward` opens one local TCP socket against one
Pod or Service and demands you remember which local port you mapped
to which remote port, `kubefwd services -n <ns>` walks every
`Service` in the namespace, opens a port-forward to each one, and —
crucially — writes entries into your local `/etc/hosts` so that
inside your laptop, `http://api`, `http://postgres:5432`, and
`http://redis:6379` resolve to those forwarded ports as if your
laptop were a Pod inside the cluster. The application code you are
debugging keeps using its in-cluster DNS names, unmodified, and
they just work.

Multi-namespace mode (`kubefwd services -n ns-a -n ns-b`) gives you
fully-qualified short and long names (`api`, `api.ns-a`,
`api.ns-a.svc.cluster.local`) so cross-namespace clients work too.
Context flags let you forward several clusters at once into the
same hosts file with prefixes, label-selector filters scope which
Services get bridged so you do not flood `/etc/hosts` with
hundreds of entries, and a context-aware reaper removes the entries
on shutdown so a `Ctrl-C` returns the laptop to its previous state.

Because `/etc/hosts` is privileged on macOS / Linux, `kubefwd`
itself needs `sudo` (or to be granted `CAP_NET_BIND_SERVICE` +
write access to `/etc/hosts`); the README documents the
sudo-with-preserved-env recipe.

## When to pick it / when not to

Reach for `kubefwd` when you are running an application **locally**
that expects to talk to a bunch of in-cluster services by DNS name —
a microservice you are iterating on that needs the rest of the
stack (database, cache, message broker, several upstream APIs) to
respond as if it were deployed alongside them. The classic case is
"the integration test suite assumes it can reach `postgres` and
`kafka` and `auth` by short DNS name" and you do not want to maintain
N `kubectl port-forward` invocations and rewrite a config file to
point at `localhost:5432` instead of `postgres:5432`.

It also pairs nicely with [Telepresence](https://www.telepresence.io)-
style "intercept one service, run it locally" workflows where you
need everything *except* the service-under-test to behave as if
you were in-cluster.

Skip it when you only need one or two ports forwarded for a
quick poke at a Pod (`kubectl port-forward` is one command and no
sudo). Skip it on shared dev machines where mutating `/etc/hosts`
is a policy issue. Skip it when the cluster's DNS names collide
with hosts your laptop already needs to resolve to other
addresses — `kubefwd` will overwrite them for the duration of
the run, which is occasionally surprising.

## Why it matters in an AI-native workflow

Coding agents that run an integration test loop locally against
remote dependencies routinely fail at the connection-string layer:
the test code is hard-coded to `postgres:5432`, the agent rewrites
it to `localhost:54321`, the tests pass, but the change leaks into
the committed config. Standing up `kubefwd services -n dev` once
at the start of the agent session means the agent's edit-test loop
can leave the in-cluster DNS names in place; the test environment
behaves like the real one, and the diff the agent produces does not
contain spurious connection-string mutations.

## Example invocations

```bash
# Forward every Service in one namespace and bridge DNS
sudo -E kubefwd services -n dev

# Forward two namespaces; cross-namespace short names also resolve
sudo -E kubefwd services -n dev -n shared-infra

# Filter by label so only data-plane Services are exposed
sudo -E kubefwd services -n dev -l 'tier=data'

# Forward across two contexts with prefixes to avoid name clashes
sudo -E kubefwd services -c staging -n dev -p st-

# Verbose mode to see exactly which Services were forwarded
sudo -E kubefwd services -n dev -v
```

## Alternatives in this catalog

- [`stern`](../stern/) — log tail across forwarded Pods; pair with
  `kubefwd` when you also need to see what those Services log as
  the local app talks to them.
- [`k9s`](../k9s/) — interactive TUI; its `Shift-F` per-Pod
  port-forward is the right tool for one-off pokes (`kubefwd` is
  for the bulk-bridge case).
- [`kubectx`](../kubectx/) — context / namespace switcher; choose
  the right context first, then `kubefwd` against it.
- [`kubectl-tree`](../kubectl-tree/) — owner-graph view; once
  `kubefwd` exposes a Service, `kubectl tree` shows which Pods
  are actually behind it.
