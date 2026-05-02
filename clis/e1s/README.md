# e1s

> **k9s-style terminal UI for AWS ECS.**
> A single Go binary that gives you a keyboard-driven TUI over
> ECS clusters, services, tasks, containers, and task definitions
> — list, describe, exec into a running container, tail logs,
> force-redeploy, scale, and edit task definitions without ever
> opening the AWS console.
> Pinned to **v1.0.53**
> ([LICENSE](https://github.com/keidarcy/e1s/blob/master/LICENSE),
> MIT).

Source: <https://github.com/keidarcy/e1s>

## TL;DR

`e1s` is to AWS ECS what `k9s` is to Kubernetes: a fast,
modal, keyboard-first terminal cockpit. It uses your ambient AWS
credentials (env vars, `~/.aws/credentials`, SSO, IMDS), lists
clusters / services / tasks in a navigable table, and exposes the
common operational verbs as one-key actions: `e` to exec into a
container via ECS Exec, `l` to tail CloudWatch logs, `U` to force
a new deployment, `s` to update desired count, `t` to register a
new task-definition revision from an editor buffer.

## Install

```bash
# Homebrew
brew install keidarcy/tap/e1s

# Go
go install github.com/keidarcy/e1s@latest

# Pre-built binaries (darwin universal, linux amd64/arm64, windows)
# https://github.com/keidarcy/e1s/releases/latest

# verify
e1s --version    # e1s version 1.0.53
```

## License

MIT — see
[LICENSE](https://github.com/keidarcy/e1s/blob/master/LICENSE).
Permissive: safe to include in internal tooling images without
extra legal review.

## One Concrete Example

```bash
# Use whatever AWS profile / region you already have configured
export AWS_PROFILE=staging
export AWS_REGION=us-west-2

# Launch the TUI; it lands on the cluster list
e1s

# Typical session:
#   ↓ ↓ Enter            # pick a cluster
#   /api Enter           # filter to services matching "api"
#   Enter                # drill into the service -> tasks list
#   Enter                # drill into a task -> containers
#   l                    # live-tail this container's CloudWatch logs
#   e                    # ECS Exec: drop into a shell in the container
#   U                    # force new deployment of the parent service
#   t                    # edit current task definition in $EDITOR,
#                        # save -> registers new revision + updates service
#   q                    # back up one level; q again to quit

# Read-only mode for prod, when you only want to look:
e1s --readonly
```

## Niche It Fills

**The "I need to debug this ECS task right now and the AWS console
is twelve clicks deep per action" gap.** ECS has no first-party
TUI. The console is slow and modal-heavy; `aws ecs ...` CLI calls
are correct but verbose and stateless. `e1s` collapses the common
incident-response loop (find service → find task → tail logs → exec
in → re-deploy) into a single keyboard-driven view.

## Why use it

1. **Uses your existing AWS auth.** No new credentials, no new IAM
   role — it calls the same SDK your `aws` CLI uses. Works with
   SSO, assume-role chains, and instance profiles out of the box.
2. **ECS Exec is one key.** `e` opens an interactive shell in the
   selected container via SSM Session Manager. No copying task
   ARNs into long `aws ecs execute-command` invocations.
3. **Edit-task-definition flow is sane.** `t` dumps the current
   task definition into `$EDITOR` as JSON; on save it registers a
   new revision and updates the service. The single most common
   "change one env var and redeploy" loop becomes one keystroke
   plus an editor save.
4. **`--readonly` for production.** Inspect freely without the risk
   of fat-fingering a `U` (force deploy) on a live cluster.

## Vs Already Cataloged

- **Vs [`k9s`](../k9s/):** `k9s` is the same shape, but for
  Kubernetes. If your workloads are on EKS, use `k9s`. If they're
  on ECS / Fargate, use `e1s`. Same muscle memory, different
  control plane.
- **Vs the `aws` CLI:** `aws ecs ...` is scriptable and stateless;
  `e1s` is interactive and stateful (remembers where you were).
  Use both — `aws` for automation, `e1s` for "what's going on
  right now".

## Caveats

- **ECS Exec must be enabled on the service.** The `e` (exec) key
  needs `enableExecuteCommand=true` on the service plus the right
  task-role permissions and SSM agent in the image. If you've
  never enabled it, that's a one-time service update.
- **Permissions matter.** The TUI is only as powerful as the
  caller's IAM. With read-only credentials, write actions
  (`U`, `s`, `t`) will fail at the API layer; nothing destructive
  can happen above your permission ceiling.
- **One region at a time.** `e1s` operates against the region in
  `AWS_REGION` / profile config. Multi-region operators need to
  re-launch (or `AWS_REGION=… e1s`) per region.
- **No CloudFormation / no Terraform integration.** `e1s` edits
  ECS resources directly. If your task definitions are managed by
  IaC, an in-TUI `t` edit will drift until you reconcile back into
  the source of truth.
