# stu

> **S3 in a TUI** — a keyboard-driven, full-screen terminal explorer
> for Amazon S3 (and S3-compatible stores like MinIO, Cloudflare R2,
> Wasabi, Backblaze B2, Ceph RGW) that lets you browse buckets and
> prefixes like a file manager, preview text/JSON/Markdown/images
> inline, peek object metadata + version history + ACLs, copy / move
> / delete / download / upload, and watch S3 events scroll in real
> time — all without touching `aws s3 ls` or the slow web console.
> Pinned to **v0.7.6** (commit
> `39de959bc1b2d1fb5a6b4487b4a04a282a9655b9`,
> [LICENSE](https://github.com/lusingander/stu/blob/main/LICENSE), MIT).

Source: <https://github.com/lusingander/stu>

## TL;DR

`stu` (Amazon **S3** **T**erminal **U**I) is what `lazygit` is to
git, but for S3: a single statically-linked Rust binary that opens
a three-pane TUI — buckets on the left, current prefix listing in
the middle, object preview/metadata on the right — and lets you
navigate the entire object hierarchy with `j/k`, `Enter`,
`Backspace`, fuzzy filter (`/`), sort by name / size / modified
(`s`), download (`d`), copy S3 URIs (`y`), and view object
contents inline. It picks up credentials from the standard AWS
chain (`~/.aws/config` / `~/.aws/credentials` profiles, env vars,
IMDS, SSO, assume-role), supports any number of named profiles
(`stu --profile prod`), honors `AWS_ENDPOINT_URL` so it Just Works
against MinIO / R2 / Wasabi / Ceph, and never makes a write call
without a confirmation prompt. There is no daemon, no config
required for read-only browsing, and quitting drops zero state on
disk — it is the missing `mc`/`s5cmd` companion for humans who
want to *see* a bucket instead of `grep`-ing through `aws s3api`
JSON.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lusingander/tap/stu

# Cargo
cargo install stu --locked

# Pre-built binaries
gh release download v0.7.6 -R lusingander/stu  # or grab from
# https://github.com/lusingander/stu/releases/tag/v0.7.6

# Arch (AUR)
paru -S stu

# verify
stu --version    # stu 0.7.6
```

Credentials come from the normal AWS resolution chain — if
`aws s3 ls` works in this shell, `stu` works too. For an
S3-compatible endpoint:

```bash
AWS_ENDPOINT_URL=https://r2.cloudflarestorage.com \
AWS_ACCESS_KEY_ID=...  AWS_SECRET_ACCESS_KEY=... \
stu --region auto
```

## Why it's worth a slot in the zoo

The S3 console is slow, eats memory, and you have to leave the
terminal. `aws s3 ls` is a flat command that needs `--recursive`
+ piping to find anything bigger than a few hundred objects.
`s5cmd` and `mc` are great scripting tools but ergonomically
brutal for *exploration* — you cannot just "look around a bucket".
`stu` fills that ergonomic gap with a TUI that is ~6 MB, has zero
config, respects every quirk of the AWS credential chain, and
treats S3 as a filesystem the way `ranger` / `yazi` treat your
local disk. For anyone who lives in S3 — log archives, ML
datasets, static-site assets, Terraform state, backups — `stu` is
the daily driver that finally makes "let me just check that
bucket" a 2-second action instead of a 30-second context switch.

## Where it sits

- vs `aws s3 ls` / `aws s3api list-objects-v2`: `stu` is *exploratory*
  (interactive, paginated, sortable, filterable, with previews);
  `aws s3` is *scriptable* (one-shot, JSON-out, automatable). They
  are complements, not substitutes.
- vs [`mc`](https://github.com/minio/mc) (MinIO Client): `mc` is a
  scriptable CLI ("`s3` answer to GNU coreutils") with `mc ls`,
  `mc cp`, `mc mirror`. `stu` is the TUI you reach for when you do
  not yet know the path you want.
- vs [`s5cmd`](https://github.com/peak/s5cmd): `s5cmd` wins on raw
  parallel-transfer throughput (it is the fastest S3 mover in
  practice). `stu` wins on "show me what is in this prefix".
- vs `cyberduck` / `Cyberduck CLI`: GUI-based, heavyweight, slow
  on remote SSH sessions. `stu` runs over SSH into a jump host with
  zero latency.
- vs the AWS console: the console is unavoidable for IAM /
  bucket-policy / lifecycle / replication config; `stu` covers
  ~95% of "I just need to look at the data" without leaving tmux.

## Footguns

- Read-only by default for safety, but `d` (delete), `m` (move),
  `p` (put / upload) are one keystroke away once enabled — they do
  prompt, but treat the confirmation seriously, especially with
  versioned buckets where a "delete" inserts a delete-marker that
  is not free.
- Listing a bucket with millions of objects under a single prefix
  costs `LIST` API calls (1000 keys per page, $0.005 / 1000 reqs);
  `stu` paginates lazily but sorting "by size" forces a full walk.
  For very wide prefixes, narrow the path first.
- The preview pane will download the *entire* object to render it.
  If you press `Enter` on a 4 GB Parquet file, you will wait and pay
  egress. Use `i` (info / metadata) instead, which is a single
  `HEAD`.
- S3-compatible endpoints vary in feature parity. R2 lacks object
  versioning; MinIO older than RELEASE.2023-06-23 lacks
  `ListObjectsV2` continuation tokens; Backblaze B2's S3 façade
  rejects `CopyObject` across buckets. Stick to the operations the
  endpoint actually supports.
- KMS-encrypted objects (SSE-KMS) require `kms:Decrypt` on the
  caller's principal *in addition to* `s3:GetObject`. The error
  surfaced in the TUI is the raw AWS string — read it.
- `stu` does not (yet) support S3 Object Lambda, S3 Access Points
  ARNs, or Multi-Region Access Points — pass the underlying bucket
  name directly.
