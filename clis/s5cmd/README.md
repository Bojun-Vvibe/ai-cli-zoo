# s5cmd

## What it does
A **parallel S3 client** written in Go that ships as a single static
binary and is engineered specifically to saturate the network pipe to
S3-compatible object stores. Verb surface mirrors AWS CLI shape — `cp`,
`mv`, `rm`, `ls`, `du`, `cat`, `head`, `select`, `pipe`, `sync` — but
the execution model is fundamentally different: every command opens a
worker pool (default `5 * NumCPU`) that fans out per-object operations
across goroutines, batches small-object requests, reuses HTTP/2
connections, and streams multi-part uploads with configurable part size
and concurrency. The `run` sub-command takes a file (or stdin) of
s5cmd commands and runs the entire batch with a single connection
pool — the typical use is `find ... | xargs -I{} echo "cp {} s3://..."
> work.txt && s5cmd run work.txt` to migrate millions of objects in
one process. `sync` does an etag/size diff between source and
destination prefix (local↔S3, S3↔S3, including across providers /
regions / accounts via per-side endpoint flags) and only copies the
delta. Wildcards (`s3://bucket/2026/*/log-*.gz`) are expanded
server-side via paginated `ListObjectsV2`. Speaks any S3-compatible
endpoint (AWS, MinIO, Ceph RGW, R2, B2 S3 API, GCS S3-compat, Wasabi,
DigitalOcean Spaces) via `--endpoint-url` + standard AWS env vars.

## Why it's interesting
Different shape from `aws s3 cp` (Python, single-process,
boto3-throttled — typically 4–10× slower on large fleets of small
objects), from `rclone` (multi-backend Swiss-army knife but per-object
overhead higher and ergonomics tuned for filesystem mirrors, not
batch S3 ops), from `mc` / MinIO Client (excellent UX but tied to
MinIO's alias system and slower for raw throughput benchmarks), and
from `goofys` / `s3fs-fuse` (mount semantics — different problem;
pick FUSE only when you genuinely need POSIX read paths, not for
batch transfer). s5cmd is the *one binary, no Python runtime, fan-out
worker pool, batched script-replay* shape: pick it specifically for
data-engineering migrations, ML training-data prep (push 10 M PNGs
from local NVMe to S3 in minutes), nightly log archive sync,
cross-region S3↔S3 replication outside of native bucket replication,
and CI jobs that need to round-trip artifacts through object storage
on a tight budget. Do **not** pick it for managing IAM / KMS / bucket
policies (no admin verbs — use `aws` for that), for partial-byte-range
streaming reads as the primary access pattern (use the SDK directly),
or for non-S3 backends like Azure Blob / GCS-native (use `rclone` or
`azcopy` / `gsutil`).

## Niche category
S3 batch transfer — Go single-binary parallel S3 client tuned for
maximum throughput on millions of objects, replacing `aws s3` in
data-pipeline and migration contexts.

## Repo
https://github.com/peak/s5cmd

## Version pinned
`v2.3.0` (latest tagged release as of 2026-05-02, published
2024-12-16)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install peak/tap/s5cmd

# Go install (any platform with a Go toolchain)
go install github.com/peak/s5cmd/v2@v2.3.0

# Pre-built binaries for darwin / linux / windows / freebsd
# https://github.com/peak/s5cmd/releases/tag/v2.3.0

# Container
docker run --rm -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY \
  peakcom/s5cmd:v2.3.0 ls s3://my-bucket/
```

## Usage examples
```sh
# Upload a directory tree with parallelism = 64
s5cmd --numworkers 64 cp 'dist/**/*' s3://artifacts/builds/2026-05-02/

# Diff-only sync (only copy what changed) from local to S3
s5cmd sync ./public/ s3://cdn-bucket/static/

# Cross-region S3-to-S3 copy without round-tripping bytes through your laptop
s5cmd cp s3://us-east-1-bucket/dataset/ s3://eu-west-1-bucket/dataset/

# Batch replay: generate a work file, then execute with one pool
s5cmd ls 's3://logs/2026-04-*/*.gz' | \
  awk '{print "cp s3://logs/" $NF " s3://archive/" $NF}' > work.txt
s5cmd run work.txt

# Talk to MinIO / R2 / any S3-compatible endpoint
s5cmd --endpoint-url https://<account>.r2.cloudflarestorage.com \
  --profile r2 ls s3://my-r2-bucket/

# Stream the head of a large object without downloading it whole
s5cmd head s3://datalake/events/2026-05-01.parquet | xxd | head
```

## Date added
2026-05-02
