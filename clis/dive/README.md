# dive

- **Repo:** https://github.com/wagoodman/dive
- **Version:** v0.13.1 (latest stable, 2025-03)
- **License:** MIT ([LICENSE](https://github.com/wagoodman/dive/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install dive` · `go install github.com/wagoodman/dive@latest` · binary releases on the GitHub release page · `docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive`

## What it does

`dive` is a TUI for inspecting a Docker / OCI image layer-by-layer. Point it
at an image (`dive ubuntu:24.04`, `dive my-app:latest`, `dive --source podman
…`, or build-and-analyze in one shot via `dive build -t my-app .`) and it
splits the screen into: (left) the layer list with per-layer command, size,
and a running total; (top-right) the aggregated filesystem tree at the
selected layer; (bottom-right) the per-layer diff (added / modified / removed
files, marked with colour). You navigate with vim keys, toggle changed-only
view (`Ctrl+U`), filter by path (`Ctrl+F`), and watch wasted space accumulate
across layers. The bottom status bar surfaces two key numbers: **image
efficiency** (% of bytes that survive to the final layer) and **wasted
bytes** (files added in one layer and deleted/overwritten in a later one — the
classic `apt-get install … && rm -rf /var/lib/apt/lists/*` mistake split
across two `RUN`s). CI mode (`CI=true dive my-app:latest` or `dive --ci`)
exits non-zero against thresholds in `.dive-ci`, so you can fail builds when
efficiency drops below a target or wasted bytes exceed a limit.

## When to pick it / when not to

Pick `dive` when an image is "mysteriously huge" and you need to see *which
layer* and *which files* are eating the bytes — `docker history` shows you
sizes per layer but not the contents, and `docker image inspect` gives you
config but not the on-disk diff. Typical wins: catching a forgotten
`COPY . .` that drags `.git/` or `node_modules/` into the image, finding a
build-time dep installed in one layer and "removed" in another (still on
disk, just shadowed), or comparing two near-identical tags layer-by-layer to
prove a base-image bump didn't bloat anything. The CI mode pairs well with
multi-stage Dockerfiles where you want a regression alarm if someone
accidentally collapses the stages.

Skip it when you only need the final filesystem (just
`docker run --rm -it img sh` or `docker export | tar tvf -`), when the image
is already small and well-understood, or when you want runtime container
behaviour analysis (use [`ctop`](../ctop/)-style tools or the orchestrator's
own metrics — `dive` is strictly a static, build-time view). It also doesn't
help with vulnerability scanning — pair with `trivy` / `grype` for that.

## Example invocations

```bash
# Inspect a public image
dive ubuntu:24.04

# Build and immediately analyze (uses the local Dockerfile)
dive build -t my-app:dev .

# Pull from a Podman / containerd / OCI archive instead of the Docker daemon
dive --source podman my-app:latest
dive --source docker-archive ./my-app.tar

# CI gate: fail if image efficiency < 95% or wasted bytes > 10 MB
cat > .dive-ci <<'EOF'
rules:
  lowestEfficiency: 0.95
  highestWastedBytes: 10MB
  highestUserWastedPercent: 0.10
EOF
CI=true dive my-app:latest
```
