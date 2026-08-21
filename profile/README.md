# InlaySQL

**An embedded SQL database in Rust.** SQLite's model — one file, no server,
embed it anywhere — with concurrent writers, native vector and BM25 retrieval,
and a MySQL wire protocol your ORM already speaks.

[**inlaysql.github.io**](https://inlaysql.github.io) · [the engine](https://github.com/inlaySQL/inlaysql) · [benchmarks](https://github.com/inlaySQL/inlaysql/blob/main/BENCHMARK.md)

---

### Retrieval is not a bolt-on

A vector column, a full-text index and your rows live in the same file, in the
same transaction, behind the same SQL. Fusing them is one statement rather than
two queries and a client-side merge:

```sql
SELECT id, body,
       fuse(vector_score(embedding, ?), bm25_score(body, ?)) AS score
FROM docs
ORDER BY score DESC
LIMIT 10;
```

### Where it stands

| | |
| --- | --- |
| Point reads | 1.33× SQLite in WAL mode, 4.97× its durable mode |
| Concurrent writes | 7.4× SQLite at 8 writers, 0% aborted |
| Vector search | 9.5× `sqlite-vec` at the same recall |
| Hybrid search | 14–17× DuckDB and pgvector |
| Reads over the MySQL wire | 1.52× MySQL 8, both sides on the same driver |
| SQL Logic Tests | 1145, all passing |

Every number regenerates from a script in the repository, and the
[benchmarks](https://github.com/inlaySQL/inlaysql/blob/main/BENCHMARK.md)
publish the losses beside the wins — joins and indexed range scans are still
slower than SQLite, MySQL still commits faster on one connection, and over the
wire its write throughput pulls away at eight. A table that only contains wins
is advertising.

### It is experimental, and says so

The on-disk format is pre-1.0 and the policy is recreate, not migrate. The
MySQL server is plaintext and localhost-first. What does not work yet is
written down in the repository rather than discovered later:
[TESTING.md](https://github.com/inlaySQL/inlaysql/blob/main/TESTING.md) covers
what is tested and what is not, and the README's
[Next](https://github.com/inlaySQL/inlaysql#next) and
[What this is not](https://github.com/inlaySQL/inlaysql#what-this-is-not)
sections cover what is being built and what is deliberately not.

### Licence

Dual licensed: the [GNU AGPL v3.0](https://github.com/inlaySQL/inlaysql/blob/main/LICENSE),
or a [commercial licence](https://github.com/inlaySQL/inlaysql/blob/main/LICENSE-COMMERCIAL.md)
from Solution Forest Limited for use without the AGPL's obligations.

### How it is built

The core is `no_std` and `#![forbid(unsafe_code)]` — it cannot read a clock or
touch a file except through a trait. That is what lets thousands of crash and
torn-write schedules replay byte-for-byte on any machine, and it is why the
same file opens natively, in a browser over WebAssembly, and through the MySQL
server without conversion.
