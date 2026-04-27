# sqlite-vec

> Snapshot date: 2026-04. Upstream: <https://github.com/asg017/sqlite-vec>
> License files (dual): `LICENSE-APACHE`
> (sha `abbef71fa08fc8040d469ea4e6e99f0ff8edd6cb`,
> <https://github.com/asg017/sqlite-vec/blob/main/LICENSE-APACHE>)
> and `LICENSE-MIT`
> (sha `9c106bc48c760f7ed9f5b8255dea7adeef029cbe`,
> <https://github.com/asg017/sqlite-vec/blob/main/LICENSE-MIT>).
> Pinned: `v0.1.9` (2026-03-31). HEAD `5778fec` on `main`.

A **single-file SQLite extension that adds vector search to any
SQLite database** — no daemon, no separate process, no Python
runtime required. Loaded into a SQLite connection it adds a
`vec0` virtual table type and a handful of scalar functions
(`vec_distance_l2`, `vec_distance_cosine`, `vec_to_json`,
`vec_quantize_int8`, `vec_quantize_binary`); a `KNN` becomes
`SELECT rowid FROM vec_items WHERE embedding MATCH ? ORDER BY
distance LIMIT 10`. Written in pure C with zero runtime
dependencies, the extension binary is ~400 KB and runs anywhere
SQLite runs: Linux, macOS, Windows, iOS, Android, Raspberry Pi,
and the browser via WASM.

## 1. Install footprint

- **Python**: `pip install sqlite-vec` — ships the precompiled
  loadable extension for Linux / macOS / Windows in one wheel
  (~1 MB). Usage: `import sqlite_vec, sqlite3; db =
  sqlite3.connect(":memory:"); db.enable_load_extension(True);
  sqlite_vec.load(db)`. Python 3.7+.
- **Node.js**: `npm install sqlite-vec` — works with
  `better-sqlite3` and the built-in `node:sqlite`. ~1 MB.
- **CLI / Rust / Go / Datasette / Ruby / Deno / browser**:
  precompiled `.so` / `.dylib` / `.dll` / `.wasm` in every release
  tarball; load with `.load /path/to/vec0` from `sqlite3` shell.
- **Datasette plugin**: `datasette install datasette-sqlite-vec`
  — exposes vector search as a SQL function inside a
  [`datasette`](../datasette/) instance.
- Workspace footprint: a single `.sqlite` file. Vector data lives
  in the same DB as your relational data; back up = `cp foo.db
  backup.db`.

## 2. Repo, version, license

- Repo: <https://github.com/asg017/sqlite-vec>
- Default branch: `main` (HEAD `5778fec` at snapshot).
- Latest release: `v0.1.9` (2026-03-31). Releases include
  precompiled binaries for every supported platform.
- License: **Apache-2.0 OR MIT** (dual-licensed; pick either).
  License files: `LICENSE-APACHE`
  (sha `abbef71fa08fc8040d469ea4e6e99f0ff8edd6cb`) and
  `LICENSE-MIT` (sha `9c106bc48c760f7ed9f5b8255dea7adeef029cbe`).
  Author Alex Garcia maintains the project independently; no CLA,
  no carve-out.

## 3. What it actually does

Three SQL primitives:

1. **`vec0` virtual table** — `CREATE VIRTUAL TABLE vec_items
   USING vec0(embedding float[384])` declares a typed vector
   column. Insert with normal `INSERT INTO vec_items(rowid,
   embedding) VALUES (?, ?)`. Storage: contiguous flat array,
   no graph index in the OSS path (brute-force KNN scan, very
   fast on ≤1M rows on commodity NVMe; the project's roadmap
   adds optional IVF / HNSW indices).
2. **`MATCH` operator + `ORDER BY distance LIMIT k`** — the KNN
   query: `SELECT rowid, distance FROM vec_items WHERE embedding
   MATCH ? AND k = 10 ORDER BY distance`. Distance metric is
   declared on the column (`float[384] distance=cosine`).
3. **Scalar quantization** — `vec_quantize_int8(?, 'unit')` and
   `vec_quantize_binary(?)` shrink storage 4× / 32× respectively
   in exchange for a small recall hit. Useful when you want a
   million-vector index in a few hundred MB of disk.

Because it is *just SQLite*, you get every SQLite feature for free:
transactions, joins onto your relational tables (the killer combo
is `JOIN vec_items v ON v.rowid = docs.id WHERE docs.tenant_id =
?`), full-text search via the `fts5` extension in the same
connection, JSON1 for metadata, and `ATTACH DATABASE` to span
multiple files.

## 4. MCP support

**No native MCP server.** The natural shape is "wrap a SQLite
connection that has `sqlite-vec` loaded behind a few MCP tools"
(`vector_search`, `insert_document`, `delete_by_id`); the
[`fastmcp`](../fastmcp/) Python SDK gives you that in ~30 lines.
For a "Datasette → MCP-over-OpenAPI" path, run
[`datasette`](../datasette/) with `datasette-sqlite-vec` installed,
then point [`mcpo`](../mcpo/) (or any OpenAPI→MCP bridge) at the
Datasette JSON API — instant tool surface over your vector
database without writing a server.

## 5. Sub-agent model

None — `sqlite-vec` is storage. Concurrency is SQLite's: one
writer at a time (with WAL enabled), N readers in parallel
without blocking the writer. For "agent reads + writes a shared
vector store from multiple processes", run with `journal_mode =
WAL` and `synchronous = NORMAL`; for "service-shaped concurrent
ingest at scale" reach for [`qdrant`](../qdrant/),
[`lancedb`](../lancedb/), or [`weaviate`](../weaviate/) instead.

## 6. Telemetry stance

**Zero telemetry.** It is a SQLite extension — there is no daemon
to phone home. Loading the extension only registers symbols inside
your existing SQLite connection. The only network traffic is what
*you* do (download embeddings from your model provider, etc.).
This is the lowest-egress vector-search story in the catalog —
even [`lancedb`](../lancedb/) and [`chroma`](../chroma/) have
optional product analytics; `sqlite-vec` has none because there is
nothing to analyse.

## 7. Token / context strategy

Not applicable — this is a vector index, not an LLM. The relevant
knobs are dimensionality (declared at column creation,
`float[384]` for MiniLM, `float[1536]` for OpenAI
`text-embedding-3-small`, `float[3072]` for `-3-large`), distance
metric (`cosine` is the default for normalized embeddings), and
quantization level (`float32` baseline, `int8` for 4×
shrink / minor recall loss, `bit` for 32× shrink / use only as a
first-stage filter).

A pragmatic two-stage pattern: store full `float32` *and* `bit`
columns; query the `bit` column with `LIMIT 200` for a fast
candidate set, then re-rank by `vec_distance_cosine(full,
query_full)` on those 200 rows. Both stages are one SQL query.

## 8. Hot keybinds

No TUI. The everyday surface is SQL inside any SQLite host:

```python
import sqlite3, struct
import sqlite_vec

def f32(v):  # serialize a list[float] to little-endian float32 blob
    return struct.pack(f"{len(v)}f", *v)

db = sqlite3.connect("rag.db")
db.enable_load_extension(True)
sqlite_vec.load(db)
db.enable_load_extension(False)

# 1. Schema — the embedding column is typed, distance metric declared
db.executescript("""
CREATE TABLE IF NOT EXISTS docs(id INTEGER PRIMARY KEY, body TEXT, tenant TEXT);
CREATE VIRTUAL TABLE IF NOT EXISTS vec_docs USING vec0(
    embedding float[384] distance=cosine
);
""")

# 2. Insert — relational + vector together, in one transaction
with db:
    db.execute("INSERT INTO docs(id, body, tenant) VALUES (?, ?, ?)",
               (1, "Pride and Prejudice was written by Jane Austen.", "wiki"))
    db.execute("INSERT INTO vec_docs(rowid, embedding) VALUES (?, ?)",
               (1, f32([0.01] * 384)))   # your real embedding here

# 3. KNN — joined with the metadata table for filter + retrieve in one round-trip
rows = db.execute("""
SELECT d.id, d.body, v.distance
FROM   vec_docs v
JOIN   docs     d ON d.id = v.rowid
WHERE  v.embedding MATCH ?
  AND  v.k = 5
  AND  d.tenant = 'wiki'
ORDER BY v.distance;
""", (f32([0.01] * 384),)).fetchall()

for row in rows:
    print(row)
```

The whole snippet runs against `:memory:` for tests, against
`./rag.db` for laptop dev, and against the same SQLite file
mounted on a server for production — same code, different DSN.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Vector search inside the SQLite file you
already have.** No new daemon, no new wire protocol, no auth
story to set up — `sqlite_vec.load(db)` and you can `JOIN` an
ANN result against your normal relational tables in one query.
The deployment story is "ship a `.db` file"; the backup story is
"`cp` it"; the air-gap story is "there is no network call to
make". For embedded, edge, mobile, and "I just want RAG that
fits in my existing app" use cases, nothing else has a smaller
footprint.

**Weakness.** **Brute-force KNN by default** — fine to ~1M
vectors at 384-d on a fast NVMe, fine to ~100K at 1536-d, but
not the tool for a 100M-vector production retrieval workload.
HNSW / IVF indices are on the roadmap and partially landed in
0.1.x but not yet at parity with [`qdrant`](../qdrant/) /
[`lancedb`](../lancedb/) for very large corpora. **Pre-1.0** —
the `vec0` table syntax has been stable for several minor
versions but the project explicitly reserves the right to
evolve it before 1.0; pin the version. **No managed cloud** —
operating it at scale is operating SQLite at scale (one writer,
WAL, careful schema migrations).

**When to choose.**

- You want **vector search inside an existing SQLite-backed
  application** without adding a second datastore.
- You want **the smallest possible deployment footprint** —
  one binary, one file, runs on a Raspberry Pi or in the
  browser via WASM.
- You want **dual-license Apache-2.0 / MIT with zero
  telemetry** and no managed service to opt out of.
- You want to **`JOIN` an ANN result onto your relational
  tables** in one SQL query (tenant filter + metadata predicate
  + KNN, all server-side).
- You are building a **mobile / edge / desktop app** that needs
  on-device semantic search and cannot ship a vector-DB server.

**When not to choose.** You have **>10M vectors** and need
HNSW / IVF / Raft replication in production → pick
[`qdrant`](../qdrant/). You want **time-travel reads of the
vector index** for evaluator reproducibility → pick
[`lancedb`](../lancedb/). You want **server-side embedding** so
clients post text not vectors → pick [`marqo`](../marqo/) or
[`chroma`](../chroma/) server. You want a **batteries-included
RAG client** with default embedder + collection-shaped Python
API → pick [`chroma`](../chroma/) and skip the SQL layer.
