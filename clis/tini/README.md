# tini

> **A tiny but valid `init` for containers.**
> Reaps zombie processes and forwards signals so that PID 1
> inside a container behaves like PID 1 should — instead of
> the typical "shell or app eats SIGTERM and Docker stops
> waiting 10 s before SIGKILL".
> Pinned to **v0.19.0**
> ([LICENSE](https://github.com/krallin/tini/blob/master/LICENSE),
> MIT).

Source: <https://github.com/krallin/tini>

## TL;DR

`tini` is a ~1k-LOC C program designed to run as PID 1 inside a
container. It does exactly two things and nothing else:

1. **Reap zombie processes.** When any descendant process exits
   without being `wait()`ed on by its parent, the kernel re-parents
   it to PID 1. If PID 1 doesn't `wait()` it, it stays as a zombie
   forever, slowly leaking entries from the global process table.
   `tini` reaps them.
2. **Forward signals.** When the container engine sends `SIGTERM`
   to PID 1, `tini` forwards it to the actual application (and
   optionally to its whole process group), so the app can shut
   down cleanly during the 10-second grace window before
   `SIGKILL`.

It is the upstream tool that Docker eventually adopted as the
`--init` flag (`docker run --init` literally injects a static
`tini` binary as PID 1).

## Install

```bash
# Static binary (recommended for container images)
ARCH=amd64   # or arm64, armhf, ppc64le, mips64el, riscv64, s390x
curl -fsSLo /usr/local/bin/tini \
    https://github.com/krallin/tini/releases/download/v0.19.0/tini-static-${ARCH}
chmod +x /usr/local/bin/tini

# Debian / Ubuntu
apt install tini

# Alpine (already in default repo, ~10 KB)
apk add tini

# Homebrew (macOS / Linuxbrew)
brew install tini

# verify
tini --version          # tini version 0.19.0
```

In a Dockerfile:

```dockerfile
FROM alpine:3.20
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["./my-app"]
```

## License

MIT — see
[LICENSE](https://github.com/krallin/tini/blob/master/LICENSE).
21 lines, copyright Thomas Orozco. Permissive, embed-friendly,
safe to ship as a static binary inside any base image (distroless,
scratch, distro-flavoured) without contaminating downstream
licensing.

## One Concrete Example

```bash
# 1. The bug tini fixes — without it
docker run --rm -it python:3.12-alpine \
    python -c "import subprocess; subprocess.Popen(['sleep', '3600']); import time; time.sleep(2)"
# Python exits at t=2s. The `sleep 3600` child gets re-parented to
# PID 1 (the python interpreter, which is now gone) → in some
# runtimes the container stays alive holding a zombie; in others
# the engine kills the whole container and you lose the visibility.

# 2. With tini
docker run --rm -it --init python:3.12-alpine \
    python -c "import subprocess; subprocess.Popen(['sleep', '3600']); import time; time.sleep(2)"
# tini is PID 1. When python exits at t=2s, tini sends SIGTERM to
# the orphaned `sleep`, reaps it, and exits cleanly with the
# python exit code. Container exits at t=2s. No zombies.

# 3. Manual ENTRYPOINT (when you don't want to rely on `--init`)
# Dockerfile:
#   ENTRYPOINT ["/sbin/tini", "--"]
#   CMD ["./my-app"]
# Now `docker run my-image` always has tini as PID 1, even on
# runtimes that don't support --init (some Kubernetes versions).

# 4. Subreaper mode for sidecar processes
tini -s -- /entrypoint.sh
# -s = register as a "subreaper" via PR_SET_CHILD_SUBREAPER, so
# tini reaps not just direct children but all descendants — the
# right setting when /entrypoint.sh forks helper daemons.

# 5. Verbose for debugging signal handling in a misbehaving image
tini -v -v -- ./my-app
# logs every signal received and forwarded; great for finding
# "why does my Java app ignore SIGTERM in the container?"
```

## Niche It Fills

**The "PID 1 problem" gap.** Container images conventionally run
the application as PID 1. PID 1 has two kernel-imposed
responsibilities: reap zombies and handle signals. Most apps
(python, node, ruby, java, shell scripts) were never designed to
run as PID 1 and silently get one or both of these wrong. `tini`
is the smallest possible drop-in that does them correctly, so
the application never has to know it's PID 1. It's the canonical
answer for production container images that need clean shutdown
(graceful drain, flush metrics, close connections in the 10 s
SIGTERM window) without rewriting the app to install signal
handlers.

## Why use it

Three things `tini` does that the obvious alternatives don't:

1. **Single static binary, ~700 KB.** No runtime, no shared libs,
   no init system, no service manager. You can bake it into a
   `FROM scratch` image with one `COPY` directive.
2. **Forwards exit code.** When the wrapped command exits with
   code N, `tini` exits with code N. Many naive `exec` wrappers
   collapse to 0/1 and break CI gating that depends on the real
   exit code.
3. **Subreaper mode.** With `-s` (or env `TINI_SUBREAPER=1`),
   `tini` registers via `PR_SET_CHILD_SUBREAPER`, catching
   re-parented orphans even when running under a parent that is
   itself PID 1 (e.g. inside a non-trivial entrypoint script
   that already had its own wrapper).

For an LLM-CLI workflow, `tini` is the **container-init layer**
under any agent that runs inside Docker / Kubernetes (a model
runner, a sandboxed code-exec container, a per-session Jupyter
kernel). When the agent process exits or is killed, `tini`
guarantees no leaked subprocesses and a clean container exit
status — so a control plane can trust "container exited 0 = task
succeeded" instead of having to guess from logs.

## Vs Already Cataloged

- **Vs [`overmind`](../overmind/) / Procfile runners:** `overmind`
  is a multi-process *supervisor* for dev (run web + worker +
  redis from one Procfile, restart on crash). `tini` is a
  container-PID-1 *reaper*. Different layer: `tini` runs *one*
  command and exits when it exits; `overmind` runs *many* and
  keeps them alive.
- **Vs `docker run --init`:** `--init` is just "Docker injects a
  copy of tini (or its built-in equivalent) for you". Identical
  semantics. Use the flag for ad-hoc runs; bake `tini` into the
  image when you need to support runtimes that don't have
  `--init` (older containerd, some k8s clusters, podman with old
  defaults, OCI-compliant runners outside Docker).
- **Vs writing signal handlers in the app:** Correct in theory,
  fragile in practice — every language runtime has subtly
  different signal semantics (Python's GIL, Java's signal masks,
  Node's libuv handlers), and you have to re-do the work in
  every image. `tini` solves it once at the kernel-PID-1 layer.

## Caveats

- **Not a service manager.** `tini` runs *one* command. If you
  need supervision of multiple long-running processes inside one
  container, use `s6-overlay`, `runit`, or just split the
  container — `tini` will not restart your app on crash.
- **Subreaper mode requires Linux ≥ 3.4.** `PR_SET_CHILD_SUBREAPER`
  is a Linux prctl. On non-Linux container runtimes (rare —
  mostly relevant for Windows containers), the `-s` flag has no
  effect.
- **`exec` form ENTRYPOINT only.** Always use the JSON-array
  form `ENTRYPOINT ["/sbin/tini", "--"]`, never the shell form,
  otherwise Docker wraps the whole thing in `/bin/sh -c` and
  *that* shell becomes PID 1, defeating the point.
- **Tag cadence is slow.** v0.19.0 is the latest stable (2021);
  the codebase is feature-complete and security-quiet. New
  releases are rare and that's a feature, not a bug — you do
  not want surprise behaviour from PID 1.
- **No log rotation, no cgroups, no nothing else.** Deliberately.
  If you want a real init system inside a container, you've
  reached for the wrong tool — look at `s6-overlay` or
  `dumb-init` (the same niche, slightly different design).
