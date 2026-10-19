# Rust persistence and serialization: SQLite, blob encoding and wire format

Research for issue #28 (part of #1). Researched 2026-10-19. Nothing here is an ADR; it feeds the decision that will become one.

Scope: re-decide persistence (was Microsoft.Data.Sqlite, #6) and serialization (was MemoryPack/MessagePack) now that ADR-0020 moved the stack to Rust. Evaluated against ADR-0002 (versioned Part state, missing Blueprint is a hard error), ADR-0009 (Chunk is the unit of catch-up; Zone preload loads blobs off the sim thread), ADR-0010 (replication wire, stable numeric type ids, encoding "a build-time benchmark"), ADR-0014 (state hash is over the persistence bytes, so the encoding must be byte-stable), ADR-0020 (persistence I/O on separate threads behind channels, no async runtime in core).

Source labels: [crates.io] = crates.io API/registry metadata; [docs] = official docs/spec site of the owner; [repo] = the project's own README or release page.

## Recommendation (short)

1. **Persistence: `rusqlite` with the `bundled` feature, WAL mode, one dedicated writer thread that owns the `Connection`.** Chunk blobs stored as BLOB rows. Use `PRAGMA user_version` for the schema version and `application_id` as a file marker.
2. **Save blob encoding: `postcard` (serde) for Part state and world tables, with an explicit per-Part state version written by us.** Only ordered containers (`BTreeMap`, sorted `Vec`) in serialized state. Terrain cell arrays may be a plain fixed-layout little-endian byte array rather than serde.
3. **Wire format: the same `postcard` payload behind our own numeric type-id tag** (ADR-0010 registry). Do not use a second format. Revisit only if a measured need appears.
4. **Rejected for now:** `bincode` (development ceased, crate is a tombstone), `bitcode` (stable format is an explicit non-goal), `rkyv` (no schema evolution story, format only promised stable within 0.8, zero-copy gain unproven for this workload), `wincode` (young, 0.x, no stated stability policy).
5. **Needs a spike before the ADR:** measure Chunk load time on representative data for postcard vs a fixed-layout terrain array vs rkyv. The public benchmark below is not our workload.

## Findings

### SQLite and rusqlite

- Current rusqlite is 0.40.2 (2026-08-08), MIT; libsqlite3-sys 0.38.2, MIT. [crates.io]
- `bundled` compiles and links a bundled SQLite and is not a default feature. README states 3.53.2 for rusqlite 0.40.1 / libsqlite3-sys 0.38.1; minimum supported system SQLite is 3.45.3. [repo: github.com/rusqlite/rusqlite]
- Bundling matters: sqlite.org reports a WAL-reset data race (found 2026-03-03) that can corrupt a database when two or more connections write or checkpoint at the same instant; fixed in 3.51.3 and later (backports 3.44.6, 3.50.7). The bundled 3.53.2 is past the fix; a system SQLite on a host would not be guaranteed to be. [docs: sqlite.org/wal.html]
- `Connection` is `Send` and `!Sync`; `Connection::open` uses `SQLITE_OPEN_NO_MUTEX` because rusqlite enforces thread safety at compile time. This fits ADR-0020: move one connection into a persistence thread and talk to it over channels. [docs: docs.rs/rusqlite/0.40.2]
- Optional `blob` feature gives `std::io` Read/Write/Seek over BLOBs (incremental blob I/O); `backup` and `serialize` features also exist. [repo]
- WAL: one writer at a time ("since there is only one WAL file, there can only be one writer at a time"); readers and the writer do not block each other. Auto-checkpoint at 1000 pages by default. Overlapping readers can starve checkpoints and let the WAL grow. WAL does not work over network filesystems. [docs: sqlite.org/wal.html]
- Durability: with `synchronous=NORMAL` in WAL mode, commits do not sync and "might rollback following a power failure or hard reset"; only checkpoints sync. Choice between NORMAL and FULL is a durability policy for the owner; saves taken at quiescent points (ADR-0002) could checkpoint explicitly. [docs: sqlite.org/wal.html]
- Single-writer is not a constraint here: a single persistence thread is the design in ADR-0020.
- Limits: default max BLOB/row length 1,000,000,000 bytes; max database 4,294,967,294 pages (about 17.5 TB at 4 KiB pages). A Chunk blob is far below this. [docs: sqlite.org/limits.html]
- Stability: SQLite states it intends to support the product "through the year 2050" and to keep the C API and on-disk format fully backwards compatible; files are bit-for-bit identical across 32/64-bit and endianness. [docs: sqlite.org/lts.html] Header fields `application_id` (offset 68, intended for application file formats) and `user_version` (offset 60, unused by SQLite) are available for our own marker and schema version. [docs: sqlite.org/fileformat2.html]
- Blob storage in a database vs files: SQLite's own 2017 test (blobs of 8-12 KB, WAL, synchronous NORMAL) found reads about 35% faster than separate files on most platforms, with `sqlite3_blob_read()` and memory-mapped I/O faster still, and per-blob transactions much slower than grouped ones. Old test, small blobs; indicative only. [docs: sqlite.org/fasterthanfs.html]
- The state hash (ADR-0014) must therefore be taken over the encoded blob bytes we produce, never over SQLite file contents. This follows from ADR-0014 as written and is not a SQLite claim.

### Blob encodings

| Crate | Version (2026-10-19) | Licence | Updated | Notes |
|---|---|---|---|---|
| postcard | 1.1.3 | MIT OR Apache-2.0 | 2025-07-24 | serde-based, specified wire format |
| bincode | 3.0.0 (tombstone); last real 2.0.1 | MIT | 2025-12-16 | development ceased |
| rkyv | 0.8.18 | MIT | 2026-08-05 | zero-copy, validation via bytecheck |
| bitcode | 0.6.9 | MIT OR Apache-2.0 | 2025-12-18 | format stability is a non-goal |
| wincode | 0.6.3 | Apache-2.0 | 2026-10-13 | bincode-compatible, MSRV 1.89 |
| rmp-serde | 1.3.1 | MIT | 2025-12-23 | MessagePack, self-describing |

All from [crates.io] registry metadata.

**postcard**
- Wire format is specified and "considered stable as of v1.0.0 and above"; a breaking wire change would need 2.0.0 and a new spec. [docs: postcard.jamesmunns.com/wire-format]
- Not self-describing: no field names or schema in the bytes; schema compatibility between revisions "is left to end users". Structs are fields in declaration order; enums are a varint discriminant then payload; maps are a count then pairs in whatever order the serializer yields. [docs]
- Not canonical: decoders accept over-long varints (`[0x80, 0x00]` for 0), and the spec says nothing guaranteeing map order, so a `HashMap` gives no byte-stability. [docs] Consequence for ADR-0014: the encoder must only see ordered containers (already required by ADR-0020's "no random-hasher HashMap in sim state"), and the hash should be of encoder output, not of arbitrary input bytes.
- Integers are varint, `f32/f64` fixed little-endian. Float use in sim state is already banned by ADR-0014.
- Consequence for ADR-0002: because the format is positional, adding or reordering a field changes the bytes. Versioned Part state therefore needs our own version tag per Part and an explicit migration function from vN to vN+1, and enums are append-only.

**bincode**
- The 3.0.0 release is a tombstone: README says development has ceased and no further releases will be published, and the crate's `lib.rs` is a compile error. It recommends wincode, postcard and rkyv. [docs: docs.rs/crate/bincode/3.0.0] Not suitable for new use.

**rkyv**
- Zero-copy; archived types are `#[repr(C)]` structs, enums use the smallest `#[repr(N)]`, objects are laid out leaves first and root last. Endianness (default little), alignment (default aligned) and relative-pointer width (default 32-bit) are cargo features that change the bytes. [docs: rkyv.org/format.html]
- Safe access needs the `bytecheck` validation feature (default on); `access_unchecked` is unsafe. [repo]
- Version policy: the 0.8.0 release notes say data from 0.7 or earlier is not compatible with 0.8, and 0.8 data is meant to stay compatible for the 0.8 line. A bump to the next minor would be another migration. [repo: github.com/rkyv/rkyv/releases] I did not find a documented schema evolution mechanism in the README, format page or FAQ; treat as absent.
- `ArchivedHashMap` defaults to `FxHasher64`; docs do not state whether layout is reproducible, so byte-stability for hashing is unproven for hash-based archived maps. [docs: docs.rs/rkyv/0.8.18] Using sorted vectors avoids this.
- The zero-copy benefit applies to Zone preload (ADR-0009) only if blobs are read in place; with SQLite that means reading the BLOB into an aligned buffer first (alignment would need care), so the saving is in avoiding a decode, not avoiding a copy. This is my inference, not a source claim.

**bitcode**
- README lists "Stable format across major versions" as a non-goal and says cross-language compatibility is also a non-goal. [repo: github.com/SoftbearStudios/bitcode] A replay segment and save are long-lived (ADR-0014 golden replays per format version), so this disqualifies it for saves. Possible for the wire if both ends are the same build, which the handshake enforces (ADR-0010), but a second format is a cost.

**wincode**
- Bincode-compatible output (README example asserts equal bytes; fuzzing harness compares round trips), owned by anza-xyz, Apache-2.0, README says release v0.1.2 had three independent security reviews. States no stability or versioning policy, and no determinism or schema-evolution claims. MSRV 1.89 per crates.io. [repo: github.com/anza-xyz/wincode] Worth watching; too young to commit saves to.

**Hand-written writers**
- ADR-0010 and ADR-0014 allow them. Given the Part count this would be a large amount of code to keep in step with the Part set; use only for hot, simple structures such as terrain cell arrays.

### Benchmarks

- The only broad primary benchmark found is `rust_serialization_benchmark`, maintained by volunteers (including the rkyv author, so not independent for rkyv), last updated 2026-10-05, nightly rustc on a 4 vCPU EPYC VM. Datasets: log, mesh, minecraft_savedata, mk48. bincode is not in it. Page calls itself a work in progress. [repo: github.com/djkoloski/rust_serialization_benchmark]
- Selected numbers (as of that run): log dataset, serialize/deserialize: postcard 1.1.3 415.6 us / 2.34 ms, size 724,953 B; rkyv 0.8.18 251.6 us / about 1.6-1.9 ms (variant results), size 1,011,488 B; bitcode 0.6.9 146.2 us / 1.46 ms, size 703,710 B; wincode 0.6.2 191.4 us / 1.82 ms, size 1,045,784 B. Mesh dataset (6 MB): postcard 514.6 us / 6.55 ms; rkyv 199.2 us / 182.6 us (variant); wincode 150.2 us / 191.1 us.
- Reading: postcard is slower than the others, notably for large numeric arrays (mesh deserialize 6.55 ms vs about 0.2 ms), but all are in milliseconds for megabyte payloads. The game is turn-based, low rate (ADR-0010), so wire speed does not matter; Chunk load matters only for Zone preload on a worker thread (ADR-0009). Large dense arrays (terrain cells) are the case where a plain byte array is better than postcard, which supports storing terrain separately from Part state.

## Open questions and gaps

- No primary source gives a postcard-vs-rkyv figure on data shaped like Chunks (sparse Entities plus dense terrain). The spike in recommendation 5 settles it.
- rkyv schema evolution: not documented in the sources I could fetch; the rkyv book may have more. Not verified.
- `synchronous` mode (NORMAL vs FULL) and checkpoint policy are a durability decision for the owner.
- Whether Chunk is stored one-per-row, or a Zone as one row plus Chunk rows, is a schema design question for the persistence ticket, not answered by sources.
- wincode and postcard `postcard-schema` (schema hashing for compatibility checks) were not evaluated in depth; the latter might help detect accidental format change in CI.
- Not covered: compression (zstd 0.14.3 BSD-3-Clause and lz4_flex 0.14.0 MIT are current per crates.io), encryption (rusqlite offers `sqlcipher` builds), and the content-pack migration ticket itself.
