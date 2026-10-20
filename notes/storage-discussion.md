# Storage discussion

Working notes for a small storage project of Ruoqi's own, grown out of Stretch's store.
Now in its own repo, TinyFrames (2026-10-20); decisions are recorded in `docs/design.md`.
Discussion only so far: nothing has been built for it yet.

**Roles:** Ruoqi designs and decides, and reviews. Claude writes all the code.

## Design philosophy (2026-10-06)

Ruoqi's film-roll metaphor comes from two physics ideas:

1. **The block universe** (eternalism; mainly from *relativity*, Minkowski spacetime): every
   moment already exists. Reality isn't a movie being played but a whole roll of film where
   every frame exists; humans experience it as an ongoing movie.
2. **Time isn't linear:** actions in the present shape the past. (In physics: Wheeler's delayed
   choice, retrocausal interpretations of *quantum mechanics* such as the transactional
   interpretation and the two-state vector formalism.) This one is mostly about the memory
   layer: forgetting and remembrance.

**How they map:**
- **Storage = the block.** All frames exist at once; the store has no "now". A reader (a client
  cursor tailing forward) is what experiences time as a movie.
- **No shared "now" across devices.** Relativity of simultaneity ↔ disagreeing device clocks;
  only causal order is invariant in both. Lamport's *Time, Clocks, and the Ordering of Events*
  (1978) was explicitly inspired by special relativity; HLCs descend from it.
- **Memory layer = the observer.** Frames never change, but their meaning does: a frame recorded
  now can be *about* the past and reinterpret it (needs both time axes: an argument for `at` in
  the storage header). The earlier understanding survives too, so the past is re-seen, not
  overwritten. That's remembrance.
- **Shredding** is the only operation where the present truly removes part of the past: a scene
  cut from the block. It's deliberate, rare, and memory-layer policy.

**Principle: the storage is the block; the memory layer is the observer.**

### Claude's response, verbatim (2026-10-06)

That's a lovely foundation, and both ideas map onto the design more precisely than metaphors usually do.

One small correction to keep the physics accurate: the block universe (eternalism) comes mainly from **relativity**, not quantum mechanics. Minkowski spacetime treats all of time as one existing four-dimensional block. The second idea, the present shaping the past, is where quantum interpretations come in: Wheeler's delayed-choice experiment, and the retrocausal interpretations such as the transactional interpretation and the two-state vector formalism.

#### Block universe → the storage layer

- **The store has no "now".** Every frame exists at once, and nothing in the store is "playing". A **reader** experiences time as a movie: a client with a cursor, tailing forward. "Now" is a property of whoever is watching, not of the film.
- **There's also no single shared "now" across devices.** In relativity, observers in different frames don't agree on which distant events are simultaneous. Only causal order (what's inside whose light cone) is the same for everyone. Distributed systems have exactly the same problem: device clocks disagree, and only "happened-before" is reliable. That's not a coincidence. Leslie Lamport's 1978 paper *Time, Clocks, and the Ordering of Events*, which founded logical clocks, was explicitly inspired by special relativity. The hybrid logical clocks in our design are a direct descendant. Your reels are, quite literally, a small spacetime.

#### The present reshapes the past → the memory layer

- **The frames never change, but what they mean does.** A frame recorded today can be *about* last Tuesday: "that wasn't procrastination, I was burnt out". It reinterprets the past without editing it, like a later measurement deciding how an earlier observation is read.
- **This is why two time axes matter.** "What happened on Tuesday, *as understood on Wednesday*" versus "*as understood today*" is a question about recorded time and about-time together. Supporting it needs both axes. That's a good argument for putting `at` in the storage header: it's how the present points back at the past.
- **The earlier understanding survives too.** You can still ask what the agent believed about Tuesday before it learned more. The past isn't overwritten; it's re-seen. That's remembrance.
- **Shredding is the one real exception.** It's the only operation where the present actually removes part of the past: a scene cut out of the block. It makes sense that it's deliberate and rare, and that it belongs to the memory layer's policy rather than happening on its own.

That gives the project a clean principle: **the storage is the block, and the memory layer is the observer.** The block is immutable and complete, with no "now". The observer moves through it, reinterprets it, and occasionally chooses to forget.

I've added this to the notes as a "Design philosophy" section near the top, with the correction about where the physics comes from. I also noted that it argues for keeping `at` in the header, since that's the open question it bears on.

## Where we are

Stretch's store (`src-tauri/src/store.rs`, design in `docs/storage.md`) is an append-only log
of whole-day JSON images plus a periodic snapshot. Ruoqi calls it "WAL-flavoured". The log is
the store: there are no data pages behind it, so it's closer to Redis AOF + RDB than to a
database WAL.

The requirements are Ruoqi's, from her design discussion that became the original CLAUDE.md:
- no database dependency
- whole-day physical records ("log outcomes, not inputs")
- `F_FULLFSYNC` durability
- fail loudly on corruption, never silently drop history
- a single writer, and no second copy that can be written back

"Small and boring" is Claude's inference (one user, a few writes a minute), not a stated
requirement.

## 2026-10-05: what "append, then sync" means

- **Append:** `write_all(line + "\n")` on an `O_APPEND` file. When it returns, the line is only
  in the kernel's buffer cache. It can be split across several `write(2)` calls, so the newline
  is the commit mark.
- **Sync:** Rust's `File::sync_all()` is `fcntl(F_FULLFSYNC)` on macOS, which flushes the
  drive's cache too. Plain `fsync(2)` on macOS stops at the drive's cache.
- **Measured on the dev Mac** (500-byte append + sync, 100 runs, median): no sync 0.00 ms,
  `fsync` 0.02 ms, `F_FULLFSYNC` 4.00 ms, Rust `sync_all` 4.00 ms. That confirms Rust uses
  `F_FULLFSYNC`.
- **Gap found and fixed (uncommitted at the time of writing):** a new file's directory entry
  wasn't synced when `stretch.log` was first created (or the data directory itself). The store
  now syncs the directory, or its parent, on creation.
- **Can't be covered by software:** a drive that reports a flush it didn't do.

## 2026-10-05: existing engines, if we didn't write our own

| | Ours | SQLite (WAL mode) | redb |
|---|---|---|---|
| Durability on macOS | `F_FULLFSYNC` per write | equal only with `fullfsync=ON` + `checkpoint_fullfsync=ON` + `synchronous=FULL`; plain `fsync` by default | `F_FULLFSYNC` via Rust's standard library (to verify) |
| Recovery | ours, ~400 lines and 11 tests | SQLite's, extensively tested | redb's, younger |
| History of changes | built in: images since the last checkpoint + 2 segments | needs a versions table | needs a versions table |
| Queries | none (all in memory) | SQL | key lookups and range scans |
| Per-record checksum | none | not by default | yes (as understood) |
| Read by hand | `cat` / `jq` | `sqlite3` | needs code |
| Dependencies | `serde_json` | bundled C | pure Rust |
| Compaction | full snapshot, O(all) | incremental | incremental |
| Second app instance | unprotected | file locks | file locks |

Also considered:
- **LMDB:** robust, but built for read-heavy, large data.
- **Ruled out:**
  - `tauri-plugin-store`: one JSON file; per-write sync not verified.
  - sled: unmaintained.
  - RocksDB / fjall: built for scale far beyond ours.
  - hosted backends: bring back the network and multiple copies.
- **CRDTs** (Automerge, cr-sqlite) only matter if Stretch ever syncs across devices.

**Triggers to switch away from ours:**
- earlier-day views or stats that want queries
- the snapshot rewrite getting slow
- a second writer (two app instances, a widget, a CLI)
- storage bugs costing more than a dependency would

Migrating would be easy: replay the log into one row per date.

## Proposed direction: a Bitcask-style engine in its own crate

The current store is already most of Bitcask:

| | Ours now | Bitcask |
|---|---|---|
| Writes | append the whole value, then sync | same |
| Index | values in memory | keys in memory → (file, offset, len); values on disk |
| Old data | full snapshot rewrite | **merge**: copy only live values from old segments |
| Startup | snapshot + replay | **hint files** (key → offset) |
| Integrity | newline + JSON parses | **CRC per record** |

Milestones, in order:
1. **Extract the crate.** Byte-oriented API: `open / put / get / delete / scan / sync`.
   Stretch becomes a client that stores JSON. No behaviour change; the tests move with it.
2. **Record framing.** `[len u32][crc32c u32][seq u64][key_len][key][value]`. A CRC failure
   at the tail is a torn write; anywhere else it's corruption.
3. **Fault-injection filesystem.** The biggest investment, because it's how storage engines
   really get tested.
   - All I/O goes through a filesystem trait; a simulated filesystem tracks synced vs unsynced
     state.
   - Crash = keep only synced data plus arbitrary prefixes of unsynced writes.
   - Inject torn writes, failing `fsync` (the "fsyncgate" lesson: after a failed fsync the page
     cache can't be trusted), and lost directory entries.
   - Run thousands of random workload + crash schedules against an in-memory reference model,
     asserting every acknowledged write survives.
4. **Keydir and hint files.** Values move out of memory.
5. **Merge.** Write, sync, rename, sync the directory, with its own crash-window analysis.
6. **Single-process lock** (`flock`).
7. **Optional:** group commit; `F_FULLFSYNC` vs `F_BARRIERFSYNC` latency vs guarantees.

Later: ordered keys and range scans (a sorted in-memory table + SSTables, toward an LSM tree),
and MVCC snapshots for readers.

These stay in Stretch, not the engine:
- the ordered write queue
- the single writer
- the no-cache rule
- "log outcomes, not inputs"

## 2026-10-05: history options, and option 3 in detail

Plain Bitcask's merge keeps only the latest value per key, which loses the history Stretch's log
keeps today. Three options:
1. **Latest value only** (pure Bitcask). History becomes the app's job.
2. **The last N versions per key.** Merge becomes version-aware; `get` gains "as of".
3. **A versioned log plus a latest-value index.** Nothing is merged away.

**Option 3: the log is the database, never rewritten.** Like Datomic, Kafka with unlimited
retention, event sourcing, or git objects.

- **Layout:** immutable sealed segments (`<first-seq>.seg`) plus one active segment;
  `index.ckpt` = `{ seq: N, key → pointer to latest version }`.
- **Frame:** `[len u32][crc32c u32][seq u64][ts u64][prev ptr][key][value]`. `prev` points to
  the key's previous version, so each key's history is a backward linked list through the log.
- **put:** append with `prev` = the current latest, sync, update the index. **get:** index
  lookup + one `pread`. **get_as_of(seq):** walk `prev` until `seq ≤ target`.
  **history:** walk the chain. **scan:** ordered index (`BTreeMap`) range. **delete:** a
  tombstone; history kept.
- **Recovery:** load `index.ckpt`, replay segments after N with CRC checks. A torn tail is
  truncated; a CRC failure elsewhere is corruption.
- **The checkpoint holds pointers, not values:** the log is the only copy of the data, and the
  index can always be rebuilt from the log. This is great for testing: compare the incremental
  index against a rebuilt one after every simulated crash.
- **Compaction becomes archiving:** sealed segments can be compressed, checksummed and moved.
  There's no merge. Checkpoint = write `.tmp` + sync + rename + sync the directory, with no
  rotation coupled to it.
- **Costs:**
  - space grows forever (~20–35 KB/day now, ~10 MB/year before compression);
  - as-of reads are O(versions walked);
  - full images per version are wasteful (deltas are a later trade-off);
  - **no true deletion without a purge** that rewrites segments, which brings merge back,
    rarely and deliberately;
  - a cold start without a checkpoint reads the whole log.
- **What Stretch gets:** "my day as planned at 9 am", undo, and outcomes kept forever.
- **Open:** design purge in from the start? It decides whether `prev` can be a raw segment
  offset, or needs to be a stable record ID that survives rewrites.

## 2026-10-05: the vision — a personal agent's life log

**Ruoqi's vision:** a simple storage backend for a personal agent she talks to constantly. Its
memory should always have a layer of "at time T on day D, she did / told me / planned X",
keyed on timestamps. It should become distributed, because she'll use the agent from several
devices, some of them offline at times. The files serve one person; even at scale, everyone
gets their own set. Her metaphor: a huge roll of film that's the story of a person's life.

**What this implies:**
1. **The unit is an immutable event (a frame),** not a whole-day image:
   `{ id, at, recorded_at, device, kind: did|said|planned|observed|inferred, content, refs }`.
   Corrections are new frames (retract/amend), never edits. This reverses CLAUDE.md's
   whole-day-image decision for the concrete reason it asked for: with several writers, day
   images conflict and force last-writer-wins, the merge that lost data in the web version.
2. **Bitemporal:** `recorded_at` (when the frame was shot) vs `at` (when the thing happens:
   plans point forward, recollections back). Like XTDB and Datomic.
3. **One reel per device** (a single-writer append-only log each; storage.md's rules apply per
   reel). The story is the splice of all reels. Sync = swap version vectors, send missing
   suffixes (like Hypercore, Secure Scuttlebutt, CouchDB). Merge = set union, a grow-only set
   CRDT: no storage conflicts. Disagreements about meaning (3 pm vs 4 pm) are both kept and
   resolved by the agent when it reads them.
4. **HLC timestamps** for ordering across devices with drifting clocks. Identity = `(device,
   seq)`; order key = `(hlc, device)`. Late footage from an offline device lands in the past,
   so the time index is separate from append order and must accept out-of-order inserts.
5. **Per-person isolation:** each person has their own reels and indexes, with no cross-person
   queries. A server is just another replica, ideally an end-to-end-encrypted,
   ciphertext-only relay.
6. **Forgetting = crypto-shredding:** a key per frame or per day, and destroying the key on
   every replica makes the data unreadable without rewriting any reel. This answers option 3's
   purge question.
7. **Agent memory = derived, rebuildable indexes** (time on both axes, text and embeddings,
   daily and weekly summaries). The agent's own conclusions are `inferred` frames with
   references to their sources ("log outcomes, not inputs").

**Scale:** ~50k words/day of transcripts ≈ 300 KB/day ≈ 100 MB/year ≈ a few GB over a
lifetime, before compression. It fits on a phone; no sharding within one person.

**Possible path:**
1. A single-device reel engine (option 3, time-keyed, CRC frames, HLC, fault-injection
   filesystem). Stretch is the first producer, writing planned/did events instead of day
   images.
2. Two-replica sync with simulated partitions; any sync order converges.
3. Encryption and crypto-shredding; a ciphertext-only relay.
4. Agent indexes (time ranges on both axes, then text and embeddings), all rebuildable.

## 2026-10-05: decided — the reel holds extracted frames, not transcripts

**Ruoqi's answer:** something in the middle. Raw conversation would be a lot, and agents already
tend to remember everything a user says. Since this store is one component of the agent's
memory, aim it at timestamp-keyed extracted facts. Keep "both", but the "raw" layer is itself a
compacted, processed version of what she said, and inferred frames exist on top.

```
raw conversation          not stored here (the chat app's, or discarded)
        │  extraction
        ▼
said / did / planned      gist frames: her words, condensed
        │  reasoning
        ▼
inferred                  the agent's conclusions, referencing gist frames
```

**Consequences:**
1. **A gist frame is an outcome:** record its provenance, `source: { session, span }` and
   `extractor: { model, version }`, so a bad gist can be traced and old extractors trusted less.
2. **Testimony vs belief:** a gist frame says "she said she plans to call Mom Sunday", not "she
   will". Testimony is permanently true (it belongs on an immutable reel); beliefs are
   `inferred` frames that can be superseded. Never mix "you told me" with "I assumed".
3. **Inferred frames churn:** each revision is a new frame superseding the last; "current
   beliefs" is a derived view (the latest frame in each chain). The reel keeps how the agent's
   understanding of her developed.
4. **Duplicates are normal** (the same thing said on two devices): storage keeps both (union);
   consolidation is an inferred frame referencing them.
5. **One reel per writer,** not per device: a server-side agent process that writes inferences
   gets its own reel.
6. **Size:** gist frames ~100–300 B; a few hundred a day ≈ 50–100 KB/day ≈ 20–40 MB/year.

**Frame sketch:**
```
{ id: (writer, seq), hlc, recorded_at, at?,
  kind: said | did | planned | inferred | retract,
  content,
  source: { session, span }?,     // said/did/planned
  extractor: { model, version },
  refs: [frame ids], confidence? } // inferred
```

## 2026-10-05: decided — storage is agent-agnostic; memory policy lives above it

**Ruoqi:** how an agent compacts and forgets is a separate layer working with this storage. The
storage is just a timestamp-keyed store, and it's useful even with no agent writing to it.

**The line: the storage provides mechanisms, clients bring policy.**

```
┌ agent memory layer (one possible client) ─────────────────────────────┐
│ kinds: said/did/planned/inferred · testimony vs belief · extractor     │
│ provenance · confidence · supersede chains · consolidation · what to   │
│ forget and when                                                        │
└──────────────────────────────── payload is opaque to the store ───────┘
┌ storage: a personal, time-keyed, replicated reel store ───────────────┐
│ frame = { id: (writer, seq), hlc, at?, group?, payload: bytes } + CRC  │
│ append · read by time range · get by id · tail/subscribe · sync ·      │
│ shred(group)                                                           │
└────────────────────────────────────────────────────────────────────────┘
```

- **The storage knows:** identity and order (`(writer, seq)`, HLC), time ranges, tail/subscribe
  (how clients maintain their own derived views), and `shred(group)` (crypto-shredding across
  replicas).
- **It doesn't know:** kinds, testimony vs belief, extractors, refs, confidence, which frames
  are current. All payload.
- **Two meanings:**
  - *Compaction*: the storage's is physical (compress sealed segments, rebuild indexes) and
    never drops a frame; the agent's is consolidation (summary frames that supersede older
    ones).
  - *Forgetting*: the storage's is `shred(group)`; the agent's is policy (what to shred, or
    just stop reading superseded frames, which needs nothing from the store).
- **Other clients with no agent:** Stretch blocks, a journal, health readings, location — any
  personal timeline.
- The earlier frame sketch (agent fields) is now an example **payload**, not the storage format.

## 2026-10-19: frames have space — reels are time, a frame's inside is space

**Ruoqi's extension:** the first use is a log of "X happens at T", but *frame* is an extendable
concept. The reels are time; inside one frame there's (in principle) unlimited space. A frame
can hold any data: visual detail, pictures, videos starting at that timestamp. Outside a frame,
frames are looked up on reels by time; inside a frame, information is found by other kinds of
organisation. Open: do videos span frames?

**In the physics frame:** a frame is a *slice* of spacetime, everything one writer captured at
one instant. Time is the axis between slices; space is the extent inside one. A video isn't a
slice at all: it's a world-tube, something with extent in time.

**What it implies for the engine:**
1. **Two addressing schemes.** *Between frames:* the store's time index (reel, `(writer, seq)`,
   HLC, `at`). *Inside a frame:* a path or structure inside the frame (`photos/2.jpg`,
   `video#t=00:12:30`, `region(x,y,w,h)`). The store provides one generic inner structure; any
   richer organisation (faces, places, embeddings) is a client index over the parts.
   Mechanisms vs policy again.
2. **Split the frame header from its contents.** Gigabytes can't go in the log: replay, CRC
   per frame, sync latency and sync between devices all assume small records. So:
   - the **log** holds the frame header plus a **manifest**: `path → (content hash, size,
     media type)`, with small parts inline;
   - a **content-addressed blob store** holds the bytes (`hash → blob`, chunked), like git
     objects and trees, or Perkeep's blobs.
   - The manifest's hashes make the frame's CRC cover its contents transitively (a Merkle
     tree). A blob's hash is its own integrity check.
3. **Commit order = blobs first, then the frame.** Write and sync the blobs, then append and
   sync the frame; appending the frame is the commit point. A crash in between leaves orphan
   blobs, which garbage collection removes: no frame refers to them. That's the same order git
   uses (objects before refs).
4. **Sync separates, too.** Frames sync eagerly (small: the story everywhere); blobs sync
   lazily or on demand (a phone keeps the reels but fetches a video when opened). Content
   addressing makes blob transfer idempotent and deduplicated.
5. **Shredding meets dedup.** Encrypting blobs with their shred group's key means the same
   photo in two groups is stored twice: deduplication across groups would need convergent
   encryption, which leaks "these two are equal" and makes shredding one copy pointless while
   the other key lives. Recommendation: dedup only within a shred group.

**Videos spanning frames — two needs, both supported:**
- **Meaning: an interval.** A video *covers* `[from, to]`. That argues for `at` being an
  interval, not a point: `at: { from, to? }`. Looking up "what was happening at 3:12 pm"
  becomes an interval query (an interval tree on `at`), landing in the video's frame, then a
  timecode lookup inside it (`t = 3:12 pm − from`). One frame, no splitting needed for meaning.
- **Durability: chunks.** A two-hour live recording can't be one atomic append: a crash at
  1:59 would lose everything. A long capture is written as a chain of segment frames (say a
  minute each, like HLS segments), each committed on its own and linked by `refs` (or a
  stream id). Readers see one video; the store sees committed slices.
- So: **a frame is an instant of recording (on the reel), whose content can describe a
  span (`at`).** Short media is one frame with an interval; long or live media is a chain.

**Effect on the milestones:** the first milestone ("X happens at T" frames, small payloads)
doesn't change, but the frame format should already have a payload *variant*, `inline(bytes)`
vs `manifest(parts)`, with only `inline` implemented. Adding the blob store later is then a
new feature, not a format migration.

**Prior art worth reading:** Perkeep (Camlistore) — personal lifetime storage built on
content-addressed blobs and mutable "permanodes"; git's object model (blobs, trees, commits);
Datomic's split of the log from storage; HLS/DASH segmenting for chunked media.

## 2026-10-19: is this a time-series database?

**Ruoqi asked.** Short answer: it shares a TSDB's *storage shape* but not its *data model*.

- **Shared:** time is the primary access path; writes are append-heavy and mostly in order;
  data lives in immutable, time-ordered blocks (Prometheus seals 2-hour blocks, like our sealed
  segments); range queries skip blocks by their min/max time; late data is a known pain
  (Prometheus only gained out-of-order ingestion in 2.39).
- **Different:**
  - *Payload:* a TSDB stores numeric samples in named series (metric + tags) and compresses
    them hard (Gorilla delta-of-delta and XOR). Frames are opaque, heterogeneous, possibly
    huge, with an inner structure.
  - *Queries:* a TSDB aggregates (rate, avg, percentiles over windows). This store retrieves
    frames; meaning comes from clients.
  - *Retention:* a TSDB downsamples and drops old data by design. This store keeps everything
    and forgets only by deliberate shredding.
  - *Time:* a TSDB has one time axis. This store is bitemporal (`recorded_at` vs `at`) with
    per-frame identity and provenance (`(writer, seq)`).
  - *Topology:* a TSDB is a server ingesting from many sources. This is one person's
    replicated reels, merged as a set union across devices.
- **Closer kin:** event stores (EventStoreDB, event sourcing), bitemporal databases (XTDB,
  Datomic), replicated append-only logs (Hypercore, Secure Scuttlebutt), personal stores
  (Perkeep).
- **Worth borrowing from TSDBs:** seal segments by time window (per day or week) and keep
  min/max time per segment, so range scans prune segments and shredding a period maps to whole
  segments. Keep a separate index for `at`, which arrives out of order. Numeric clients (health
  readings) can pack Gorilla-style series inside a frame's payload — the engine needn't know.

## 2026-10-20: it's a memory palace

**Ruoqi's observation:** "this is a memory palace." The reels-are-time, frames-are-space model
is the method of loci: a fixed route of places, each holding something to remember, recalled
by walking the route and looking around each room.

**The mapping:**
- **Route = reel.** The order you walk is time. Unlike a classical palace, the route isn't
  chosen in advance: it grows at the end every moment, and time picks the loci.
- **Room = frame.** One locus; its inside is space, where things are *placed*.
- **Loci never move = immutability.** A palace works because the rooms stay as you left them.
  Frames are never rewritten; a later reinterpretation is a new frame pointing back (`refs`,
  bitemporal `at`), like going back to a room and leaving a new object beside the old one.
- **Images are cues, not the whole memory = inline vs manifest.** The frame carries the small,
  vivid cue; the heavy material sits in the blob store, fetched when you walk in.
- **Associations between rooms = cross-frame references.** Palaces rely on links ("the key in
  the kitchen opens the cellar"); that's `refs` and client indexes.
- **Forgetting on purpose = crypto-shredding.** Destroying a key is bricking up a room.

**What it suggests for the design:**
1. **Navigation is the primary read, query the secondary.** "Walk the reel, step into a frame,
   look around" is `scan(range)` → `open(frame)` → `list(parts)`. That argues the scan path
   and frame opening must be the fastest things the engine does; ad-hoc queries are client
   indexes on top.
2. **Spatial organisation inside a frame is a real candidate**, not just paths. A part could
   carry an optional position (region, or a coordinate in some client-defined space), so
   "what's in this corner of the room" is answerable. Still opaque: the store keeps the
   coordinates, clients give them meaning. Feeds the open question on a frame's inner
   structure.
3. **Landmarks.** Palace walkers anchor on memorable loci. Sparse, client-chosen markers on a
   reel (a day, a trip, "the week I moved") are cheap to support as a small index of
   `(at range, name, frame refs)` — and Stretch's day is already one.

**Where the analogy breaks:** a palace is a technique for a lossy store (human memory) to
recall; this store doesn't forget unless told to. The palace's value here is as a *reading*
model — how a person moves through the data — not as a durability model.

**Refined (Ruoqi, same day):** TinyFrames *plus a remembrance layer on top* is the memory
palace, not TinyFrames alone. That fits the earlier split (storage is agent-agnostic; memory
policy lives above it):
- **TinyFrames is the building.** Rooms (frames) on a route (reels), never moved; durable,
  fast to walk; no opinion about what matters.
- **The remembrance layer is the art of memory.** It decides what to place and where (which
  cue goes inline, what position inside a frame), lays the associations (`refs`, indexes),
  names the landmarks, guides the walk (what to surface when, rehearsal, "on this day"),
  and decides what to let go (when to shred). All of that is policy, written as new frames
  or held in rebuildable indexes, never as edits to old ones.
- This also resolves "where the analogy breaks": the lossy, selective part of a palace
  belongs to the remembrance layer; the store underneath stays lossless.
- **Test for the boundary:** if a feature is *how* to remember (what's salient, what links to
  what, what to recall now), it's remembrance-layer; if it's *keeping* (append, order,
  durability, sync, shred mechanics, scan/open/list), it's TinyFrames.

## Open questions

- **Raw retention** *(agent layer, not storage)*. Is the raw conversation kept anywhere, even for a while (e.g. 30 days on
  the device, or encrypted outside this store)? If not, gist frames can never be re-extracted
  by a better model, and `source` is informational only.
- **`at` in the storage header or the payload?** Header = the store keeps a second time index
  any client can query (Stretch planned vs done, calendars, recollections); payload = the store
  stays recorded-time-only and clients index `at` themselves by tailing. Claude leans header
  (optional field); it doubles the engine's indexing work. The design philosophy (the present
  reinterprets the past) argues for the header.
- **Shred group granularity.** Per frame, per day, per session, or chosen by the client?
- **Name and scope.** *Answered (2026-10-19):* **TinyFrames**, a general personal timeline
  store (one person's reels), not Stretch-only. The frames may not be tiny; the name is about
  the engine aiming to be fast and simple. Project instructions: `CLAUDE.md` at the repo root.
- **Values.** *Answered:* opaque bytes; schemas belong to clients.
- **History.** *Implied by the vision:* option 3 (the store never drops a frame; forgetting is
  shredding). Not explicitly confirmed by Ruoqi.
- **Purge.** Answered by crypto-shredding as a storage mechanism; when to use it is client
  policy.
- **API style.** Sync only, or an async front for Tauri commands?
- **Testing depth.** How far to take the fault-injection model (directory reordering? partial
  sector writes?).
- **`at` as an interval** *(2026-10-19)*. Media and other spanning content argue for
  `at: { from, to? }` rather than a point. Strengthens the case for `at` in the header.
- **The inner structure of a frame** *(2026-10-19)*. Flat paths (`photos/2.jpg`), a nested
  tree, or typed parts? How much does the store define, and how much is client index?
  *(2026-10-20, memory palace)*: should a part carry an optional spatial position?
- **Blob store scope** *(2026-10-19)*. Chunking scheme (fixed size vs content-defined, as in
  restic/borg), garbage collection of orphans, and which devices hold which blobs.
- **Dedup vs shredding** *(2026-10-19)*. Confirm dedup only within a shred group.
- **Landmarks** *(2026-10-20, memory palace)*. Is a small index of named `at` ranges
  (a day, a trip) a store index, or purely client-side?
- **Stretch's existing log.** Migrate `stretch.log` into the new format via a one-time replay.
