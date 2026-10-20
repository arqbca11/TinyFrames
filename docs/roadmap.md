# TinyFrames roadmap

Status as of 2026-10-20: design only, no code yet. Decisions are in `docs/design.md`; the
reasoning is in `notes/storage-discussion.md`.

Each milestone lists what it delivers, how we know it's done, and which open questions
(`CLAUDE.md`) must be settled first.

## 1. Single-replica reel engine

**Delivers:** a Rust library crate with a byte-oriented API: `open / append / get(id) /
range(time) / tail / sync`. CRC-framed records, `inline` payloads only, HLC, sealed segments
plus an active segment, checkpoint, `flock` per reel.

**Done when** these tests pass (temp dir per test):
- append + reopen gives the same frames; `seq` continues across restarts
- a torn final record is truncated and recovery succeeds; a record without its full length is
  torn
- a CRC failure in the middle fails loudly and leaves the files untouched
- checkpoint then replay equals a full replay
- a crash between checkpoint rename and anything after it
- opening in a data directory that doesn't exist yet
- a second writer on the same reel is refused
- out-of-order HLCs (late frames) are found by time-range reads
- the rebuilt index equals the incremental index

**Blocked on:** `at` in the header or the payload (and point vs interval), since it fixes the
record format. Also: sync-only vs async API; the CRC32C dependency.

## 2. Fault-injection filesystem

**Delivers:** all I/O behind a filesystem trait; a simulated filesystem tracking synced vs
unsynced state. A crash keeps synced data plus arbitrary prefixes of unsynced writes. Injected
torn writes, failing `fsync`, lost directory entries.

**Done when:** thousands of random workload + crash schedules run against an in-memory
reference model, asserting every acknowledged frame survives and the rebuilt index equals the
incremental one. Failures reproduce from a seed.

**Blocked on:** testing depth (directory-entry reordering? partial sector writes?); the
property-testing dependency.

## 3. Stretch as the first client

**Delivers:** Stretch writes planned/did events instead of day images. The existing
`stretch.log` is migrated with a one-time replay. Stretch keeps its own rules (ordered write
queue, single writer, no write-back cache, "log outcomes, not inputs").

**Done when:** Stretch runs on TinyFrames with its history intact, and its old store is
removed.

**Blocked on:** Stretch's event payload schema (a Stretch decision, not a TinyFrames one).

## 4. Two-replica sync

**Delivers:** version-vector exchange and reel-suffix transfer between replicas.

**Done when:** under simulated partitions, any sync order converges to the same set of frames
on every replica.

## 5. Encryption and crypto-shredding

**Delivers:** per-group frame encryption, `shred(group)` across replicas, a ciphertext-only
relay.

**Blocked on:** shred group granularity; dedup vs shredding (affects milestone 6).

## 6. Manifests and the blob store

**Delivers:** the `manifest(parts)` payload; a content-addressed, chunked blob store; blobs
committed before frames; orphan GC; lazy blob sync.

**Blocked on:** a frame's inner structure (flat paths, tree, typed parts; optional spatial
position per part); chunking scheme; which devices hold which blobs.

## 7. Client indexes

**Delivers:** rebuildable indexes: time ranges on both axes, then text and embeddings.
Possibly landmarks (named `at` ranges) if they belong in the store.

**Blocked on:** whether landmarks are a store index or purely client-side.

## Later, optional

- Group commit.
- `F_FULLFSYNC` vs `F_BARRIERFSYNC` (latency vs guarantees), measured on the target Mac.
- Compression of sealed segments.
