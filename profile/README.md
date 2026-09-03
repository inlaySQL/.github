<div align="center">
  <img src="../assets/inlaysql-square.svg" width="160" alt="InlaySQL">
</div>

# InlaySQL

**An embedded SQL database in Rust.** SQLite's model — one file, no server,
embed it anywhere — with concurrent writers, native vector and BM25 retrieval,
and a MySQL wire protocol your ORM already speaks.

[**inlaysql.github.io**](https://inlaysql.github.io) · [the engine](https://github.com/inlaySQL/inlaysql) · [benchmarks](https://github.com/inlaySQL/inlaysql/blob/main/BENCHMARK.md) · [framework examples](https://inlaysql.github.io/frameworks/) · [Chrome extension](https://github.com/inlaySQL/inlaysql-chrome)

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

Over the MySQL wire it also answers to MySQL's own spelling: `MATCH (body)
AGAINST (?)` is the native BM25 probe, not an emulation of one, and
`CREATE FULLTEXT INDEX` is the DDL Laravel's `$table->fullText()` already
emits. Boolean mode is refused by name rather than quietly answered as if it
were natural-language mode.

### Where it stands

| | |
| --- | --- |
| Point reads | 692,893 ops/s, ~4× SQLite's durable mode; p50 625 ns |
| Concurrent writes | 1,184 commits/s, ~13× SQLite at 8 writers, 0% aborted |
| Vector search | 68.67 µs, ~9× `sqlite-vec` at 100% recall |
| Hybrid search | 192.0 µs, ~60-70× DuckDB and pgvector |
| Indexed range scan | 119,219 ops/s, ~8× MySQL 8.4 and ~5.5× PostgreSQL 17 |
| Reads over MySQL wire | 10,292.4 ops/s, ~1.2× MySQL 8.4 at 1 connection |
| SQL Logic Tests | 1307, all passing |

Every number regenerates from a script in the repository, and the
[benchmarks](https://github.com/inlaySQL/inlaysql/blob/main/BENCHMARK.md)
publish the losses beside the wins. Batch inserts beat MySQL 8.4 and lose
to PostgreSQL 17 like for like (0.68×), and range scans and the `LIMIT 10` join shapes lose
to SQLite — while `GROUP BY` now beats both servers and the full two-table
joins win 3× and 8× against SQLite. MySQL commits faster on one connection and pulls further ahead at
sixteen. A table that only contains wins is advertising.

These multiples are rounded to the precision the harness's own measured
run-to-run spread supports — repeating the identical binary against identical
data moves them by several percent, and the benchmarks say by how much.

### It is experimental, and says so

The on-disk format is pre-1.0 and the policy is recreate, not migrate. The
MySQL server is localhost-first: it binds 127.0.0.1 by default, the account
model is still one credential from a flag, and the wire is plaintext until
`--tls-cert`/`--tls-key` are given (`--tls-required` then refuses any login
that did not encrypt). Without a certificate, do not put it on a network you
do not own. What does not work yet is
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
