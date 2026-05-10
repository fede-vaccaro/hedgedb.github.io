# Sorted String Tables in HedgeDB

## SST File Layout

Each SST file is structured this way:

```text
┌──────────────────────────────────────┐
│  index blocks                        │  ← array 4 KiB pages, contains sorted key values
├──────────────────────────────────────┤
│  second level index                  │  ← one key per index block page
├──────────────────────────────────────┤
│  filter                              │  ← probabilistic membership filter
├──────────────────────────────────────┤
│  footer                              │  ← offsets, key count, epoch, partition
└──────────────────────────────────────┘
```

Once flushed, SST files are **immutable** and can only be deleted after being **merged**.

### Second level Index

The **second level index** is an array of sorted keys. Each index block of
the core index is represented from its greatest key, thus there is a 1-to-1
positional mapping between a second level index entry and an index block: a `lower_bound()` binary search allows to identify the block a record *could* resides into.


### Approximate Probabilistic Membership filters

HedgeDB employs **Quotient filters**. Quotient filters can be more memory
expensive than the traditional Bloom filter, for instance, but they have
good **spatial locality**, hence less RAM hops are needed, making it faster
at scale.

The filter is also pinned in memory.

### Point lookup algorithm

The **SST Lookup flow** is pretty straightforward:

```text
1. Probe filter
   - If false return KEY_NOT_FOUND
2. Binary search the second level index
3. Identify the index block page
4. Fetch page from disk
5. Linear scan looking for the key.
```

In order to guarantee a single page load (1-random IO Operation) on point lookup, the invariances are mantained:

- The second level index is always pinned in primary memory.
- **Key uniqueness**: Within a single SST, a key is unique.

Without the latter, it cannot be guaranteed that a second-level-index entry maps to a single index block. **Key Uniqueness** has also implications on the MVCC implementation strategy.

## Index Block layout

HedgeDB uses 4 KB index-blocks, and the block format implements prefix
compression with restart points.

Within a block, key-values are sorted by lexicographic key order (`memcmp`
equivalent). A restart group is a subset of key-values where the first key
is serialized with no compression; for each following key, the encoder
stores how many bytes of the prefix are shared (if any) with the previous
key.

For example, consider this group of 4 keys:

```text
#0 "0ffa-1524"
#1 "0ffa-3134"
#2 "0ffa-3111"
#3 "0fff-1010"
```

It gets compressed to

```text
#0 - "0ffa-1524" -> "0ffa-1524" <- restart group leader
#1 - "0ffa-3134" -> "3134" <- shares "0ffa-" prefix of length 5 with key #0
#2 - "0ffa-3111" -> "11" <- shares "0ffa-31" prefix of length 7 with key #1
#3 - "0fff-1010" -> "a-1010" <- shares "0ff" prefix of length 2 with key #2
```

This is particularly convenient with the common practice of 24-byte keys
formed by concatenating a `partition key` (a 16-byte UUID or hash) and an
8-byte `clustering key`: the `partition key` forms a repeated pattern that
compresses well under this scheme.

The prefix compression strategy implies that a record **cannot** be found
with random access; the preceding keys within the **same restart group**
must be decoded first. Only the **restart group leader** (the first key)
does not need decoding.

The middle-ground solution that HedgeDB applies is to use **multiple
restart groups** within the same block.

Finally, at the end of the index block, the following information is stored:

- The offsets pointing to every restart group leader (2 bytes each)
- The offset count (2 bytes)
- The number of key-values stored (2 bytes)
- The block checksum (`xxhash`)

### Key Partitioning (or sharding)

HedgeDB was built around the problem of storing uniformly distributed keys.

Instead of one global LSM tree, HedgeDB shards the key space across
**16 independent partitions** by default (`2^4`, supported up to `2^16`).
Each partition has:

- Its own level hierarchy (L0, L1, L2, ...)
- Its own `shared_mutex` for lock isolation
- Its own `manifest` file persisting the level structure

**Why partition?** Compaction and flushes on different partitions run
**fully in parallel** with zero lock contention.

### How Keys Map to Partitions

The partition is determined by the **first two bytes of the key**,
interpreted as a big-endian `uint16_t`:

```cpp
prefix = (k[0] << 8) | k[1]   // range [0x0000, 0xFFFF]
```

With 16 partitions (default), the 16-bit space is divided into bands of
width 4096. Each partition is identified by the **upper-bound prefix** of
its band:

| Key prefix range | Partition ID |
|------------------|--------------|
| 0x0000-0x0FFF    | 0x0FFF       |
| 0x1000-0x1FFF    | 0x1FFF       |
| ...              | ...          |
| 0xF000-0xFFFF    | 0xFFFF       |

:::{important}
**Design constraint.** If keys aren't uniformly distributed over their
first two bytes (e.g. all keys share a prefix), partitions become
unbalanced, and the imbalance cascades: compaction and flush parallelism
degrade, level-size targets within an overloaded partition fall behind, and
the per-SST quotient filter sizing drifts off-target. HedgeDB is optimized
for UUIDs, hashes, or other uniformly-distributed keys; if your keys don't
satisfy this, hash or salt them before insertion.
:::

### On-Disk Layout

Each partition's files live in a directory structure like:

```text
partitions/01/013f.000042    ← SST file for partition 0x013F, flush sequence 42
partitions/01/manifest       ← Serialized level structure for fast reload
```

The `manifest` file is newline-delimited, with one line per level and
semicolon-separated filenames within a level. This lets the database reload
quickly on startup without re-scanning all SSTs.

## K-Way Merge

The merge procedure (`src/db/merge/merge.cc`) stream-reads from the input SSTs
and also stream-writes into the newly merged SST, combining the key-values with a
k-way merge based algorithm that:

- deduplicates keys (newest epoch wins) (keys are **unique** within a single SST);
- drop key-values if the value is a tombstone and the merge is to the _bottommost level_;


## MVCC and Transaction Isolation in HedgeDB

HedgeDB provides read consistent snapshots through **Multi-Version
Concurrency Control** (MVCC).

During a point lookup or range scan, a **transient snapshot** of the database
is taken, providing read-consistency. HedgeDB provides snapshot isolation for reads only; there is no transaction API.

More in detail:

- at **memtable level**, a global **Sequence Number** (SeqNo) is
maintained, which is incremented on every `put` operation (insertion,
update, or delete). When a range iterator is initialized, the current SeqNo
is atomically loaded, and every record with a SeqNo greater than the
snapshot is skipped, so the user does not see it at all.

- at the **SST level** (and block level), only the highest SeqNo is serialized into the SST footer: during flush, only the latest version of a key is written and the other are discarded. When merging multiple SSTs, if a key has multiple versions, only the most recent is kept thus preserving the **key uniquess** property. The range iterator acquires a full snapshot of the SSTs belonging to a specific partition. Once flushed, any SST is **immutable**, the reader has a consistent view over the stored records. At runtime, SSTs and the underlying file descriptors are reference counted by their observers, and the removal from filesystem is deferred until the reference count drops to zero, i.e. until the last observer has finished.

This approach, result of the **immutability** and **key uniqueness** properties of the SSTs and reference counting, is quite straightforward and also keeps the guarantee of the single-page-lookup per SST but comes with a downside: it creates transient space amplification until the user is done with the range scan operation.

## Durability in HedgeDB

HedgeDB persists every `put` and `delete` operation on the Write-Ahead Log. The WAL goes through the page-cache thus it offers protection under application crash events. However,it does not provide `fdatasync` options yet, hence it does not shields against power-loss events or system crashes.

Structural changes have stronger guarantees: these are always `fdatasync`-ed. Structural changes include Manifests updates, and SST creation (both the SST file and the directory are `fdatasync`-ed).