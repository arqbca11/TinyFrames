# TinyFrames — a personal, time-keyed, replicated frame store

The frames themselves aren't necessarily tiny (a frame can hold a video). The name is about
the engine: **fast and simple.** Prefer the smallest mechanism that keeps the durability rules;
every feature, file format field, dependency and code path has to earn its place. When two
designs are equally correct, pick the one with less code and fewer moving parts, and say what
the other would have bought.

A storage engine for one person's timeline: an append-only, never-rewritten log of **frames**
on **reels**, looked up by time, replicated across that person's devices. The first use is a log
of "X happens at T". The long-term client is a personal agent's memory: "at time T on day D,
she did / told me / planned X". Any personal timeline can use it, with or without an agent:
a day planner (Stretch, its first client), a journal, health readings, photos, location.

It began as Stretch's store (`stretch.log` + snapshot, in the Stretch repo:
`src-tauri/src/store.rs`, design in `docs/storage.md`) and grew from there. It's a learning and
portfolio project as much as a dependency.

## Where the design came from

The reasoning behind this file is in **`notes/storage-discussion.md`**, the design discussion
that started in the Stretch repo and moved here with this project. It's a dated record of
the discussion:
- the design philosophy;
- existing engines considered;
- history options;
- the personal-agent vision;
- the "frames, not transcripts" and "agent-agnostic storage" decisions;
- frames having space;
- the comparison with a time-series database;
- TinyFrames plus a remembrance layer as a memory palace;
- the open questions as they stood.

- **Read it** before proposing a change to anything under "Decided" or settling an open
  question, so a choice isn't re-argued without the reasons it was made.
- **This file wins** where the two disagree: the discussion records how ideas evolved, and
  some of its early directions (e.g. the Bitcask-style key-value engine) were superseded.
- **Keep it going.** When Ruoqi discusses design, add a dated section (`## YYYY-MM-DD: topic`)
  with her idea, the analysis and any recommendation, and update its open questions. Don't
  rewrite earlier sections; mark a superseded idea as superseded in place.
- **Discussion vs decisions:** the notes hold the thinking; once Ruoqi decides something,
  record the decision (date, reason) in `docs/design.md` and update this file's "Decided" or
  "Open questions" to match.

## Working with Ruoqi

- **Ruoqi designs, decides and reviews. Claude writes all the code.** Don't ask her to write or
  edit code by hand; propose, implement, and explain.
- She's a software engineer with a database internals background. Explain storage and systems
  choices precisely (sync semantics, crash windows, invariants) and skip the basics.
- **Decisions are hers.** Frame design questions as options with trade-offs and a
  recommendation, then wait. Don't settle an item listed under "Open questions" by
  implementing it; ask.
- Keep `docs/design.md` current: when a decision is made or changed, record it there with the
  date and the reason, and update the open questions.

## The philosophy

The reel is a film roll. Two ideas from physics shape the design:

1. **The block universe** (relativity, Minkowski spacetime): every moment exists at once. The
   store is the block: complete, immutable, with **no "now"**. A *reader* (a client cursor,
   tailing forward) is what experiences time as a movie.
2. **The present reshapes the past** (Wheeler's delayed choice; retrocausal interpretations of
   quantum mechanics): frames never change, but their meaning does. A frame recorded today can
   be *about* last Tuesday and reinterpret it, while the earlier understanding survives. The
   past is re-seen, not overwritten. That's remembrance, and it lives in the layer above.

Also: **no shared "now" across devices.** Device clocks disagree the way observers'
simultaneity does; only causal order is reliable (Lamport 1978 was inspired by relativity).

And: **reels are time; a frame's inside is space.** Between frames you navigate by time. Inside
a frame there is, in principle, unlimited space (text, pictures, video), navigated by structure.

**Principle: the storage is the block; the memory layer is the observer.**
**Rule: the storage provides mechanisms; clients bring policy.**

## Layers

```
┌ clients (policy) ───────────────────────────────────────────────────────┐
│ agent memory: kinds (said/did/planned/inferred), testimony vs belief,   │
│ extractor provenance, confidence, supersede chains, consolidation,      │
│ what to forget · Stretch: planned/done blocks · journal · health · …    │
└───────────────────────────────────── payload is opaque to the store ────┘
┌ the store (mechanisms) ─────────────────────────────────────────────────┐
│ frames on reels · append · get by id · read by time range · tail /      │
│ subscribe · sync between replicas · shred(group)                        │
└─────────────────────────────────────────────────────────────────────────┘
```

**The store knows:** frame identity and order, time ranges, tailing, sync, shredding.
**It doesn't know:** what a frame means, kinds, refs between frames, which frames are current.
All of that is payload, and clients build their own derived views by tailing.

## Decided

1. **The unit is an immutable frame.** Corrections are new frames (retract / amend /
   supersede, all client-level), never edits. Whole-state images (Stretch's whole-day records)
   are not the model: with several writers they conflict and force last-writer-wins, which is
   exactly the merge that once lost Stretch's data.
2. **The log is the database, never rewritten.** Sealed immutable segments plus one active
   segment. No merge, no compaction that drops frames. "Compaction" means physical work only:
   compressing sealed segments, rebuilding indexes. (Like Datomic, event sourcing, git
   objects.)
3. **One reel per writer.** Each reel is a single-writer append-only log. A writer is a
   device, or a process such as a server-side agent writing inferences. The full story is the
   splice of all reels.
4. **Identity and order.** Frame id = `(writer, seq)`, `seq` strictly increasing per reel.
   Order key = `(hlc, writer)` with hybrid logical clocks, so drifting device clocks still
   respect causality. Late frames from an offline device land in the *past*, so time indexes
   are separate from append order and accept out-of-order inserts.
5. **Sync is set union.** Replicas exchange version vectors and send missing reel suffixes
   (like Hypercore, Secure Scuttlebutt, CouchDB). A grow-only set: no storage-level conflicts.
   Disagreements about meaning are both kept; readers resolve them.
6. **Forgetting is crypto-shredding.** Frames are encrypted with a per-group key; destroying
   the key on every replica makes them unreadable without rewriting any reel. When to shred is
   client policy.
7. **Per-person isolation.** Each person has their own reels and indexes; no cross-person
   queries. A server is just another replica, ideally an end-to-end-encrypted, ciphertext-only
   relay.
8. **Payloads are opaque bytes.** Schemas belong to clients.
9. **The agent stores extracted "gist" frames, not raw transcripts** (agent-layer decision;
   noted here for sizing): ~100–300 B per frame, a few hundred a day, ≈ 20–40 MB/year. Media
   is the exception and goes to the blob store (below). No sharding within one person.

## Frames and their space

- A frame is an **instant of recording** on its reel. Its content may describe a **span**
  (a video covers `[from, to]`).
- **Header vs contents.** The log holds the frame header plus its payload. The payload is a
  variant, decided now so adding media later isn't a format migration:
  - `inline(bytes)` — small payloads, stored in the log. **Milestone 1 implements only this.**
  - `manifest(parts)` — `path → (content hash, size, media type)`, small parts inline; the
    bytes live in a **content-addressed blob store** (`hash → blob`, chunked), like git
    objects or Perkeep blobs. The manifest's hashes make the frame's CRC cover the contents
    transitively (Merkle).
- **Commit order for media:** write + sync blobs, then append + sync the frame. The frame
  append is the commit point; orphan blobs from a crash are garbage-collected.
- **Long or live media** is a chain of segment frames (about a minute each, like HLS), each
  committed on its own and linked by the client, so a crash loses at most the last segment.
- **Sync splits:** frames sync eagerly (small); blobs lazily or on demand.
- **Inside a frame** the store offers one generic structure (paths in the manifest). Richer
  organisation (faces, places, embeddings, timecodes) is client indexing over parts.

## On disk (single replica)

```
reels/<writer>/<first-seq>.seg    sealed segments, immutable
reels/<writer>/active.seg         the active segment (one writer)
index.ckpt                        checkpoint: { per-reel seq, indexes → pointers }
blobs/                            later: content-addressed blob store
```

**Frame record (proposed, not final):**

```
[len u32][crc32c u32][seq u64][hlc u64][at?][group?][payload kind u8][payload]
```

A segment is sealed by size or by time window, and keeps its min/max time so range scans can
skip whole segments (borrowed from time-series databases).

**Indexes are derived and rebuildable.** The checkpoint holds pointers, never values: the log
is the only copy of the data. After any crash the index can be rebuilt from the log, which is
also the main test oracle (incremental index == rebuilt index).

## Durability rules (inherited from Stretch's store; non-negotiable)

- **Append, then sync, then acknowledge.** One `write_all` of the record, then `sync_all()`
  — on macOS that's `fcntl(F_FULLFSYNC)`, which flushes the drive's cache; plain `fsync(2)`
  stops at the drive's cache. (Measured on Ruoqi's Mac: `fsync` 0.02 ms, `F_FULLFSYNC`
  4.00 ms per 500-byte append.) Never acknowledge an unsynced write.
- **Sync the directory** after creating, renaming or deleting any file (new segment, new data
  directory via its parent, checkpoint rename).
- **A failed append is truncated back**; if that fails too, the store is poisoned until
  restart.
- **After a failed sync, don't trust the page cache** (the "fsyncgate" lesson): poison and
  recover from disk.
- **Recovery:** load the checkpoint, then replay records after it with CRC checks. A torn final
  record (short, or CRC failure at the very tail) is truncated. A bad record anywhere else is
  **corruption: stop, leave the files untouched, surface the error. Never silently drop
  history.**
- **Checkpoint:** write `.tmp`, `sync_all`, `rename`, sync the directory.
- **One writer per reel per process:** take an `flock` on the reel; a second opener fails.
- Software can't cover a drive that lies about flushing. Say so; don't pretend otherwise.

## Milestones

Detail (deliverables, done-when, what blocks each) is in `docs/roadmap.md`.

1. **Single-replica reel engine.** Rust crate, byte-oriented API: `open / append / get(id) /
   range(time) / tail / sync`. CRC-framed records, `inline` payloads only, HLC, segments +
   checkpoint, `flock`. Stretch's store tests carry over in spirit.
2. **Fault-injection filesystem** — the biggest investment. All I/O goes through a filesystem
   trait; a simulated filesystem tracks synced vs unsynced state. A crash keeps only synced
   data plus arbitrary prefixes of unsynced writes. Inject torn writes, failing `fsync`, lost
   directory entries. Run thousands of random workload + crash schedules against an in-memory
   reference model, asserting every acknowledged frame survives and the rebuilt index equals
   the incremental one.
3. **Stretch as the first client.** Stretch writes planned/did events instead of day images.
   Migrate the existing `stretch.log` with a one-time replay. (Stretch keeps its own rules:
   the ordered write queue, the single writer, no cache that can be written back, "log
   outcomes, not inputs".)
4. **Two-replica sync** with simulated partitions; any sync order converges.
5. **Encryption and crypto-shredding**; a ciphertext-only relay.
6. **Manifests and the blob store** (media).
7. **Client indexes** (time ranges on both axes, then text and embeddings), all rebuildable.

Later, optional: group commit; `F_FULLFSYNC` vs `F_BARRIERFSYNC` (latency vs guarantees);
compression of sealed segments.

## Stack

- **Rust**, a library crate first; any binary (CLI, server, relay) comes later.
- **Minimal dependencies. Ask before adding any.** Likely candidates to propose: a CRC32C
  crate, `serde`/`serde_json` for checkpoints and tests, a property-testing crate for the
  fault-injection runs. No embedded database (SQLite, redb, sled, RocksDB): this *is* the
  database.
- Target macOS on Apple Silicon first. Durability claims must name the platform they were
  measured on.

## Tests that must exist (from milestone 1)

Use a temp dir per test, and the simulated filesystem once it exists.

- append + reopen gives the same frames; `seq` continues across restarts
- a torn final record is truncated and recovery succeeds; a record without its full length is
  torn
- a CRC failure in the middle fails loudly and leaves the files untouched
- checkpoint, then replay from it gives identical state to a full replay
- a crash between checkpoint rename and anything after it
- opening in a data directory that doesn't exist yet
- a second writer on the same reel is refused
- out-of-order HLCs (late frames) are found by time-range reads
- the rebuilt index equals the incremental index

## Not this project's job

Meaning. Kinds, refs, supersede chains, confidence, summaries, which frames are "current",
what to forget and when — all client policy. Also: queries beyond id, time range and tail;
aggregation (it's not a time-series database: it shares a TSDB's storage shape, not its data
model or its retention); multi-person anything.

## Open questions

- **`at` in the header or the payload?** Header = the store keeps a second time index any
  client can query (planned vs done, recollections, media spans). Claude leans header,
  optional, and as an **interval** `{ from, to? }` (media argues for it; so does the
  philosophy). It doubles the indexing work.
- **Shred group granularity:** per frame, per day, per session, or the client's choice?
- **Dedup vs shredding:** dedup only within a shred group (cross-group dedup needs convergent
  encryption, which leaks equality)? Claude recommends yes.
- **A frame's inner structure:** flat paths, a nested tree, or typed parts? How much the store
  defines vs client indexes. Should a part carry an optional spatial position (a region, or a
  coordinate in a client-defined space), opaque to the store? (Memory-palace discussion.)
- **Landmarks:** sparse, client-chosen named `at` ranges (a day, a trip, "the week I moved").
  A small store index of `(at range, name, frame refs)`, or purely client-side?
- **Blob store:** chunking (fixed vs content-defined, as in restic/borg), orphan GC, which
  devices hold which blobs.
- **Does the store need stable keys / version chains** (the earlier "option 3" `prev` pointer
  per key), or are supersede chains purely payload? Leaning payload.
- **Raw conversation retention** *(agent layer)*: if raw transcripts aren't kept, gist frames
  can never be re-extracted by a better model.
- **API style:** sync only, or an async front for app integrations (Tauri commands)?
- **Testing depth:** directory-entry reordering? Partial sector writes?

## Reading list

- Lamport, *Time, Clocks, and the Ordering of Events in a Distributed System* (1978)
- Kulkarni et al., *Logical Physical Clocks* (hybrid logical clocks, 2014)
- Bitcask paper (Basho, 2010) — the starting shape
- Pillai et al., *All File Systems Are Not Created Equal* (OSDI 2014) — crash consistency
- *Can Applications Recover from fsync Failures?* (USENIX ATC 2020) — fsyncgate
- Datomic and XTDB docs — immutable logs, bitemporality
- Hypercore / Secure Scuttlebutt — single-writer replicated logs
- Perkeep (Camlistore) — personal lifetime storage, content-addressed blobs
- FoundationDB's simulation testing; TigerBeetle's VOPR — deterministic fault injection
