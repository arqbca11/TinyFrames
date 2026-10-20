# TinyFrames design record

The decisions, each with its date and reason. The thinking behind them is in
`notes/storage-discussion.md`; the working summary for contributors is `CLAUDE.md`. When a
decision is made or changed, record it here (date, reason) and update `CLAUDE.md` to match.

## Decisions

### 2026-10-05: the unit is an immutable frame
Corrections are new frames (retract / amend / supersede, all client-level), never edits.
**Why:** whole-state images (Stretch's whole-day records) conflict once there are several
writers and force last-writer-wins, the merge that once lost Stretch's data.

### 2026-10-05: the log is the database, never rewritten
Sealed immutable segments plus one active segment. No merge, no compaction that drops frames;
"compaction" is physical only (compressing sealed segments, rebuilding indexes). This was
"option 3" of the history options.
**Why:** a personal timeline's value is its history. Bitcask-style merge keeps only the latest
value per key. Forgetting is handled by crypto-shredding instead, so the purge that would
bring merge back isn't needed.
*Note:* the discussion records this as implied by the vision rather than explicitly confirmed.

### 2026-10-05: one reel per writer
Each reel is a single-writer append-only log. A writer is a device, or a process such as a
server-side agent writing inferences. The full story is the splice of all reels.
**Why:** single-writer logs need no coordination to append, and replicate as suffixes.

### 2026-10-05: identity `(writer, seq)`, order `(hlc, writer)`
`seq` is strictly increasing per reel. Hybrid logical clocks order frames across devices.
Time indexes are separate from append order and accept out-of-order inserts.
**Why:** device clocks drift and disagree; only causal order is reliable. Frames from an
offline device arrive late and land in the past.

### 2026-10-05: sync is set union
Replicas exchange version vectors and send missing reel suffixes (like Hypercore, Secure
Scuttlebutt, CouchDB). Disagreements about meaning are both kept; readers resolve them.
**Why:** a grow-only set has no storage-level conflicts, so any sync order converges.

### 2026-10-05: forgetting is crypto-shredding
Frames are encrypted with a per-group key; destroying the key on every replica makes them
unreadable without rewriting any reel. When to shred is client policy.
**Why:** it gives real deletion while keeping the log immutable.

### 2026-10-05: per-person isolation
Each person has their own reels and indexes; no cross-person queries. A server is just another
replica, ideally an end-to-end-encrypted, ciphertext-only relay.

### 2026-10-05: the store is agent-agnostic; payloads are opaque bytes
The store provides mechanisms (identity, order, time ranges, tail, sync, shred); clients bring
policy (kinds, refs, supersede chains, confidence, what's current, what to forget). Schemas
belong to clients.
**Why:** the store should be useful for any personal timeline, with or without an agent.

### 2026-10-05: the agent stores gist frames, not raw transcripts *(agent layer)*
~100–300 B per frame, a few hundred a day, ≈ 20–40 MB/year. No sharding within one person.
**Why:** raw conversation is large and agents already retain it elsewhere; this store holds
timestamp-keyed extracted facts. Recorded here for sizing.

### 2026-10-19: name and scope — TinyFrames
A general personal timeline store (one person's reels), not Stretch-only. The name is about the
engine aiming to be fast and simple; frames themselves can be large.

### 2026-10-19: the payload is a variant from the first format version
`inline(bytes)` or `manifest(parts)`. Milestone 1 implements only `inline`.
**Why:** adding media and the blob store later becomes a new feature, not a format migration.

## Proposed, not final

- **Frame record:** `[len u32][crc32c u32][seq u64][hlc u64][at?][group?][payload kind u8][payload]`.
  Blocked on the `at` question below.
- **Media commit order:** write + sync blobs, then append + sync the frame (the commit point);
  orphan blobs are garbage-collected. Long or live media is a chain of ~1-minute segment
  frames.
- **Segments** sealed by size or time window, each keeping min/max time for range pruning.

## Open questions

Kept in sync with `CLAUDE.md`. See that file for the current list and the leanings.
