# Persistence store: SQLite and alternatives (issue #6, map #1)

Question: SQLite (Microsoft.Data.Sqlite and alternatives) versus per-zone binary/MessagePack files, for a persistent world with zone-granular load/save, a world vs character save boundary, schema versioning/migration, and concurrent server access. Leaning SQLite.

## Answer

**Confirmed: use SQLite via Microsoft.Data.Sqlite, WAL mode, one writer.** Per-zone MessagePack blobs are a fine *payload* format inside it, but a bare file-per-zone store makes you rebuild atomic multi-zone commits yourself. LiteDB is the only other embedded option examined and offers nothing here that SQLite does not.

Key reasons, each sourced below:

1. Atomic commit across many rows/pages (a zone transition touching two zones plus a character) comes free; with files you must build a journal/rename protocol. [S4, S8]
2. WAL gives concurrent readers with a single writer, which matches a server-authoritative design where one simulation thread owns writes. [S2]
3. Schema versioning has a built-in slot (`PRAGMA user_version`) and file-type tag (`application_id`). The migration mechanism itself is by convention, not provided. [S1, S4]
4. Constraint: Microsoft.Data.Sqlite async is synchronous, so persistence I/O must be moved off the simulation thread by design, not by `await`. [S3]

## Facts by sub-question

### Atomicity and crash safety
- SQLite commits are atomic across a transaction, surviving crashes and power loss; the guarantee relies on a journal holding a verifiable pre-image plus a single commit point, not on writes being atomic. [S8]
- SQLite's application-file-format page claims atomic updates, page-level incremental writes (only changed parts hit disk), and that custom formats typically need whole-file rewrites. This is the SQLite authors' position and gives no benchmarks. [S4]
- Atomicity holds per database file; across ATTACHed databases it does not hold as a set in WAL mode. [S2] Implication: avoid designs needing a cross-file atomic commit (see boundary below).

### Concurrency (server access)
- WAL: readers do not block writers and vice versa; only one writer at a time. [S2]
- WAL requires all processes on the same host (shared-memory wal-index); no network filesystems. [S2] Fits listen server and dedicated server on one machine.
- WAL setting persists in the file. Auto-checkpoint at 1000 pages by default; a commit that triggers it can be slow; checkpoint starvation is possible with overlapping readers. [S2]
- `synchronous=NORMAL` under WAL stays consistent but a committed transaction may roll back after power loss/OS crash; app crashes lose nothing. `FULL` adds a WAL sync per commit. [S1] Decision input: NORMAL for routine zone autosave, FULL at moments where loss matters (e.g. character death).
- Known WAL-reset bug (rare race with two connections writing/checkpointing simultaneously) affects 3.7.0 through 3.51.2, fixed in 3.51.3 (backports 3.44.6, 3.50.7). [S2] Ensure the bundled native SQLite is at or above the fix (not verified here).
- Microsoft.Data.Sqlite: connection/command/reader objects are not thread-safe; open a connection per use (pooling on by default since 6.0). Busy/locked errors are retried automatically until the command timeout (default 30 s). [S3b, S6]
- Async ADO.NET methods run synchronously. Avoid them; use WAL. [S3]
- `Cache=Shared` with WAL is discouraged. [S6]
- Online backup exists (`BackupDatabase`) but currently blocks writers while it runs. [S7]

### Zone-granular load/save
- A `zone(zone_id PRIMARY KEY, version, payload BLOB)` table gives per-zone load/save in one transaction. Limits are not a concern: default max BLOB 1 GB (hard limit 2^31-3), max database about 17.5 TB at 4 KB pages. [S9]
- SQLite authors claim blobs under about 100 KB read/write faster than separate files. [S4] Whether zone payloads fall in that range is for a prototype to measure.

### Payload encoding (MessagePack for C#)
- Integer keys fastest; missing keys deserialize to default; add fields with new keys, never reuse removed ones. [S10]
- Source generator available; README says AOT-safe in the usual case (AOT details not read). [S10]
- LZ4 supported; measure per case. [S10]
- Untrusted-data warnings apply if saves/mods can be user-supplied: use `MessagePackSecurity.UntrustedData`, avoid Typeless. [S10]
- Hybrid option: hot/queryable fields as columns, bulk state as blob; decide when the entity model lands.

### World vs character save boundary
Design question, not dictated by a source. Both options are supported by the facts above:
- **A. Single DB file**, separate tables (`world_*`, `character_*`). One transaction can cover a zone and a character together. Portable characters need export/import.
- **B. Separate files** (world.db, characters.db). Characters portable, but no atomic commit across files in WAL [S2], so transitions need an idempotent two-step protocol.
- Recommendation: A. Character death plus the world event that spawns the next character can then commit atomically.

### Schema versioning and migration
- `PRAGMA user_version` is a 4-byte application-owned integer SQLite never reads; `application_id` is a 32-bit tag for tools like `file(1)`. [S1]
- SQLite documents backward-compatible extension by adding tables/columns but describes no migration framework. [S4]
- Implication: own a small forward-only migration runner keyed on `user_version`, run in a transaction at startup, plus a per-blob payload version (MessagePack keys handle additive change).

## Alternatives considered
- **Bare per-zone MessagePack files:** fast and inspectable, but atomic multi-file updates, crash recovery, locking and partial-write detection are yours to build ([S8] describes what that entails). Rejected as primary store.
- **LiteDB v5:** embedded .NET document store, single data file, 16 MB root-document cap, AES encryption. The docs page read did not state concurrency or maintenance status, so neither is verified. [S11] No identified advantage over SQLite here.
- **EF Core over SQLite:** same engine; enables WAL by default for databases it creates [S3]. Optional convenience, extra weight for blob-oriented storage.
- **Not researched:** sqlite-net, server databases (Postgres), RocksDB/LMDB. Excluded by the in-process/listen-server requirement; not verified.

## Not verified
- Microsoft.Data.Sqlite / SQLitePCLRaw support for .NET 11 and the bundled native SQLite version.
- Real latency of zone load/save at project zone sizes; needs a spike.
- Native AOT specifics for MessagePack.

## Sources (fetched 2026-10-18)
- S1 SQLite pragma reference (user_version, application_id, journal_mode, synchronous): https://www.sqlite.org/pragma.html
- S2 SQLite WAL: https://www.sqlite.org/wal.html
- S3 Microsoft.Data.Sqlite async limitations: https://learn.microsoft.com/en-us/dotnet/standard/data/sqlite/async
- S3b Microsoft.Data.Sqlite database errors/locking: https://learn.microsoft.com/en-us/dotnet/standard/data/sqlite/database-errors
- S4 SQLite as an application file format: https://www.sqlite.org/appfileformat.html
- S6 Microsoft.Data.Sqlite connection strings: https://learn.microsoft.com/en-us/dotnet/standard/data/sqlite/connection-strings
- S7 Microsoft.Data.Sqlite online backup: https://learn.microsoft.com/en-us/dotnet/standard/data/sqlite/backup
- S8 SQLite atomic commit: https://www.sqlite.org/atomiccommit.html
- S9 SQLite limits: https://www.sqlite.org/limits.html
- S10 MessagePack-CSharp README: https://github.com/MessagePack-CSharp/MessagePack-CSharp
- S11 LiteDB docs: https://www.litedb.org/docs/
