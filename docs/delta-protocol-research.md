# Delta Push/Pull Protocol Research: elfshaker vs Git Pack Files

## Executive Summary

elfshaker and Git both solve the problem of efficiently storing many versions of
files, but they operate at fundamentally different granularities and with different
design goals. Git stores individual objects (blobs, trees, commits) with
per-object delta compression. elfshaker stores whole files grouped into
Zstandard-compressed frames, relying on zstd's long-distance matching rather
than explicit delta encoding. A Git-style delta push/pull protocol is feasible
for elfshaker but would need to be adapted to its coarser-grained, frame-based
architecture.

---

## 1. Structural Comparison

### 1.1 Object Model

| Aspect | Git | elfshaker |
|---|---|---|
| **Unit of storage** | Individual object (blob, tree, commit, tag) | Whole file (identified by SHA-1 of content) |
| **Object identity** | SHA-1 (or SHA-256) of type+size+content | SHA-1 of raw file content |
| **Object types** | 4 types (blob, tree, commit, tag) | Single type (file content blob) |
| **Metadata** | Stored as objects (trees, commits) | Stored in `.pack.idx` (msgpack-serialized index) |
| **Versioning** | DAG of commits pointing to trees of blobs | Ordered list of snapshots with delta-encoded file lists |

Git's object model is content-addressed and forms a DAG. elfshaker is flatter:
snapshots are ordered lists of `(path, checksum, mode)` tuples, and objects are
raw file contents identified by their SHA-1.

### 1.2 Pack File Format

#### Git Pack Format (v2)

```
Header:    "PACK" + version(4B) + num_objects(4B)
Objects:   [type(3b) + size(variable) + data] ...
           type can be OBJ_COMMIT/TREE/BLOB/TAG or OFS_DELTA/REF_DELTA
Trailer:   SHA-1 of entire pack
```

- Each object is individually framed with its type and size
- Delta objects reference a base (by offset or SHA-1) and store a binary diff
- Objects can be randomly accessed via the `.idx` file (fanout table + sorted SHA-1 list + offsets)
- Delta chains can be arbitrary depth (typically capped at ~50 in practice)

#### elfshaker Pack Format

```
Header:    Zstandard skippable frame (0x184D2A50 magic)
           Contains msgpack-serialized PackHeader:
             - magic: 848629801635942891
             - frames: Vec<PackFrame{frame_size, decompressed_size}>
Frames:    [Zstandard frame containing concatenated objects] ...
Index:     Separate .pack.idx file with:
             - snapshot_tags, snapshot_deltas (ChangeSets)
             - path_pool, object_pool (EntryPools)
             - object_metadata (offset + size per object)
```

- Objects are concatenated within Zstandard frames (default: 1 frame per 512 MiB)
- No per-object framing inside a frame; objects located by offset+size from index
- Decompression is streaming/sequential within a frame (no random object access)
- Multiple frames enable parallel decompression

### 1.3 Delta/Compression Strategy

#### Git: Explicit Per-Object Deltas

Git computes binary deltas between similar objects using a modified xdelta algorithm:

1. **Base selection**: Heuristic based on path name, file size, and type
2. **Delta encoding**: Copy-from-base + insert-new-data instruction stream
3. **Delta chains**: Object A → delta(B, A) → delta(C, B) → ...
4. **Thin packs**: For network transfer, base objects can be omitted if receiver has them

The delta format is:
```
source_size (varint) + target_size (varint) + instructions:
  - Copy: 1xxxxxxx + offset(1-4B) + size(1-3B)  → copy from base
  - Insert: 0xxxxxxx + literal data               → insert new bytes
```

This achieves very high compression for similar objects (e.g., two versions of the
same source file might delta down to a few hundred bytes).

#### elfshaker: Implicit Compression via Zstandard

elfshaker does **not** compute explicit deltas between objects. Instead:

1. Objects sorted by size within a frame (grouping similar-sized files together)
2. Zstandard's long-distance matching (window up to 256 MiB) finds repeated
   byte sequences across objects within the same frame
3. Snapshot-level delta encoding: `ChangeSet<FileHandle>` stores only added/removed
   files between consecutive snapshots (index-level, not data-level)

This works remarkably well for elfshaker's use case (many builds of the same
project) because:
- Identical files across builds share the same SHA-1 and are stored once
- Files that differ only slightly end up near each other in a frame, and zstd's
  long-distance matching effectively captures the similarity
- The 256 MiB window is large enough to span many related files

---

## 2. Similarity Score

Rating the similarity on a scale of 1-10 across dimensions:

| Dimension | Similarity | Notes |
|---|---|---|
| Content-addressed storage | 8/10 | Both use SHA-1 to identify objects; elfshaker lacks Git's type prefix |
| Pack file concept | 7/10 | Both bundle objects into archives; internal structure differs significantly |
| Delta encoding | 2/10 | Git uses explicit per-object deltas; elfshaker relies on zstd implicit matching |
| Index structure | 4/10 | Both have sorted-offset indexes; Git's is a binary fanout table, elfshaker's is msgpack |
| Snapshot/commit model | 3/10 | Git has a DAG of commits; elfshaker has ordered snapshot lists with ChangeSets |
| Remote protocol | 3/10 | Git has bidirectional smart protocol; elfshaker has read-only HTTP fetch |
| Object granularity | 6/10 | Both store file-level blobs; Git also stores trees/commits as objects |

**Overall similarity: ~5/10** — The high-level concepts (content-addressed objects
bundled in packs) are shared, but the internal mechanics diverge significantly.

---

## 3. How Git's Push/Pull Protocol Works

### 3.1 Reference Discovery

Client connects and receives the server's list of refs (branches, tags) with
their commit SHA-1s.

### 3.2 Negotiation (have/want)

**Fetch (pull):**
```
Client → Server: want <SHA-1>      (for each ref the client wants)
Client → Server: have <SHA-1>      (for each commit the client already has)
Client → Server: done
Server → Client: ACK <SHA-1>       (for common ancestors found)
Server → Client: NAK               (if no common ancestor)
Server → Client: <packfile stream>
```

**Push:**
```
Client → Server: <ref-update commands>
Client → Server: <packfile stream>  (thin pack of objects server lacks)
```

### 3.3 Thin Packs

For network efficiency, Git generates "thin packs" where delta bases can be
objects the receiver already has (not included in the pack). The receiver
"thickens" the pack by adding missing bases after receipt.

### 3.4 Key Protocol Properties

- **Bidirectional negotiation** minimizes data transfer
- **Object-level granularity** means only new/changed objects are sent
- **Delta compression against known bases** further reduces transfer size
- **Incremental**: only the difference between two repository states is transferred

---

## 4. Feasibility of a Delta Protocol for elfshaker

### 4.1 What Would Need to Change

#### A. Negotiation Layer

elfshaker would need a negotiation mechanism. The natural analog to Git's
have/want is:

```
Git concept          → elfshaker analog
─────────────────────────────────────────
commit SHA-1         → snapshot checksum (already exists: compute_snapshot_checksum)
ref (branch/tag)     → snapshot tag or pack name
object SHA-1         → object checksum (ObjectChecksum, same as Git)
```

A protocol could look like:

```
Client → Server: want <snapshot_tag>
Client → Server: have_snapshot <snapshot_checksum>
Client → Server: have_objects <object_checksum> ...   (optional, for fine-grained)
Server → Client: ACK/NAK
Server → Client: <delta pack stream>
```

The `have_objects` line is the key optimization: the client tells the server
which objects (by SHA-1) it already has, so the server can exclude them from
the pack it generates.

#### B. Incremental Pack Generation

The server would need to:

1. Resolve the requested snapshot to its full file list
2. Diff against the client's known objects to find the set of missing objects
3. Generate a pack containing only the missing objects
4. Optionally: generate a partial index covering just the transferred objects

This is **directly feasible** with elfshaker's current architecture:

- `PackIndex::resolve_snapshot()` already materializes full file lists
- Object deduplication by SHA-1 is already the core mechanism
- The pack creation pipeline (`batch::compress_files`) accepts arbitrary
  iterators of object readers — it can produce a pack of any subset of objects
- A new "thin index" could be generated covering just the transferred objects,
  which the client merges into its local index

#### C. Object-Level Delta Encoding (Optional Enhancement)

Git's biggest advantage in transfer size is per-object binary delta encoding.
elfshaker could add this, but there are trade-offs:

**Option 1: zstd dictionary-based compression**
- Pre-compute a zstd dictionary from representative objects
- Ship the dictionary separately; compress transferred objects against it
- Simpler than custom delta encoding; leverages existing zstd infrastructure
- Moderate compression improvement

**Option 2: Explicit binary deltas (like Git)**
- For each new object, find the most similar known object (by path + size heuristic)
- Compute a binary diff (e.g., using `zstd --patch-from` or a dedicated library like `bidiff`)
- Transfer only the delta + base reference
- Maximum compression but significant implementation complexity
- Requires the receiver to have base objects available for reconstruction

**Option 3: zstd `--patch-from` mode**
- Zstandard natively supports using a "source" file as a dictionary for
  compressing a "target" file (`ZSTD_refPrefix()`)
- The server could compress each new object using the closest matching old
  object as a reference prefix
- This is essentially delta compression but using zstd's infrastructure
- The client would need to know which base to use for each object

**Recommendation**: Option 3 (zstd `--patch-from`) is the sweet spot. It
leverages elfshaker's existing zstd dependency, achieves near-optimal delta
compression for binary files, and avoids implementing a custom delta format.

#### D. Push Support

elfshaker currently only supports pull (HTTP GET). Push would require:

1. **Server-side write endpoint**: Accept uploaded packs and indexes
2. **Conflict resolution**: Handle concurrent pushes (simpler than Git since
   elfshaker doesn't have a commit DAG — just append new snapshots)
3. **Authentication/authorization**: Required for write access

A minimal push protocol:
```
Client → Server: push <pack_name> <snapshot_tags...>
Client → Server: have_objects <checksum> ...          (objects already on server)
Server → Client: want_objects <checksum> ...          (objects server needs)
Client → Server: <pack stream of wanted objects>
Client → Server: <index data>
Server → Client: OK / ERR
```

### 4.2 Architectural Challenges

#### Frame-Level Granularity vs Object-Level

Git can extract/transfer individual objects because each is independently
framed. elfshaker's objects are concatenated within zstd frames — extracting
a single object requires decompressing the entire frame up to that object's
offset.

**Impact on protocol**: The server must decompress frames to extract individual
objects for a thin transfer pack. This is CPU-intensive but not prohibitive —
the server would maintain a decompressed object cache or use frame-level
granularity (only send entire frames that contain needed objects).

**Possible optimization**: A "frame-level" transfer mode that sends whole frames
when most objects in a frame are needed, falling back to object-level transfer
only when few objects from a frame are required.

#### No Random Access Within Frames

Generating a thin pack requires reading specific objects from existing packs.
Since elfshaker decompresses sequentially, the server would need to:
- Decompress frames and cache/index the objects, or
- Maintain a loose object store alongside packs, or
- Use a seek-friendly secondary format for serving

#### Index Merging

After receiving a partial pack, the client must merge the new objects and
snapshot metadata into its existing indexes. elfshaker's `PackIndex` already
supports `push_snapshot()` and merging operations for the pack workflow, so
this is feasible but would need a network-aware merge path.

### 4.3 Effort Estimate by Component

| Component | Complexity | Existing infrastructure |
|---|---|---|
| Snapshot-level negotiation | Low | `compute_snapshot_checksum` exists |
| Object set differencing | Low | SHA-1 comparison, already core to dedup |
| Thin pack generation | Medium | `compress_files` exists, needs subsetting |
| Thin pack reception + merge | Medium | Index merge partially exists |
| Push endpoint (server) | High | No server implementation exists |
| zstd `--patch-from` delta | Medium | zstd crate supports `refPrefix` |
| Frame-aware transfer optimization | Medium | Frame metadata exists in PackHeader |

### 4.4 Protocol Sketch

```
=== FETCH (pull) ===

1. Client sends list of wanted snapshot tags
2. Server responds with snapshot metadata (checksums, object lists)
3. Client computes set difference: objects_needed = server_objects - local_objects
4. Client sends "have" list (local object checksums, potentially abbreviated)
5. Server generates thin pack of missing objects:
   a. For each missing object, optionally delta-compress against a base the
      client has (using zstd refPrefix)
   b. Bundle into zstd frames
   c. Generate partial index
6. Server streams: [partial_index | compressed_frames]
7. Client receives, verifies checksums, integrates into local repository

=== PUSH ===

1. Client sends list of snapshot tags to push
2. Server responds with objects it already has (or a bloom filter thereof)
3. Client computes missing objects, generates thin pack
4. Client streams: [partial_index | compressed_frames]
5. Server receives, verifies, integrates

=== OPTIMIZATIONS ===

- Bloom filter for "have" negotiation (avoids sending full object lists)
- Frame-level transfer when >50% of objects in a frame are needed
- Persistent connections for multiple operations
- Resumable transfers (frame-level checkpointing)
```

---

## 5. Comparison Summary

```
                    Git                         elfshaker
                    ───                         ─────────
Storage unit:       Object (blob/tree/commit)   File content blob
Delta encoding:     Explicit (xdelta-like)      Implicit (zstd LDM)
Pack structure:     Object stream w/ types       Zstd frames of concatenated files
Index:              Binary fanout + offsets       Msgpack pools + metadata
Random access:      Yes (per object via .idx)    No (sequential within frame)
Snapshots:          Commit DAG                   Ordered list with ChangeSets
Remote protocol:    Smart bidirectional          Read-only HTTP GET
Object negotiation: have/want on commit SHAs     Not implemented (full pack fetch)
Thin packs:         Yes (delta against bases     Not implemented
                    receiver already has)
Push support:       Yes                          No
```

---

## 6. Conclusion

A delta push/pull protocol for elfshaker is **feasible and architecturally
natural**, though it would differ from Git's protocol in important ways:

1. **Negotiation would operate at the snapshot + object level**, not at the
   commit-graph level. elfshaker has no DAG, so there's no "common ancestor"
   walk — instead, the client simply reports which objects it has.

2. **Object-level delta compression** is the largest gain available. Using
   zstd's `refPrefix` / `--patch-from` mode would be the most natural
   implementation, avoiding a custom delta format while getting comparable
   compression to Git's delta encoding.

3. **The biggest gaps** are: (a) no server-side implementation exists,
   (b) object extraction from frames requires sequential decompression,
   and (c) index merging needs a network-aware path.

4. **The biggest advantages elfshaker has** over a direct Git port:
   zstd's long-distance matching already provides implicit delta compression
   within frames, the simpler snapshot model avoids complex graph negotiation,
   and the existing `ChangeSet` mechanism provides efficient snapshot-level
   differencing.

The recommended approach would be iterative:
1. First: implement object-set negotiation and thin pack generation (no deltas)
2. Second: add zstd `refPrefix` delta compression for transfer
3. Third: add push support with a lightweight server
4. Fourth: optimize with bloom filters and frame-level transfer modes
