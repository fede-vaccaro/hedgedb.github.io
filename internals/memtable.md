# Memtable Swap and Per-Thread WAL

:::{note}
Glossary for this section: a thread performing an insert operation is
referred to as a "Writer thread" or just "Writer", and the threads
performing the flush from memtable to SST are referred to as the "Flusher
thread" or just "Flusher".
:::

## The Concurrent Skip List

**Skiplists** are a natural fit for memtables: they are **sorted associative
data structures**, are "upgradable" to support concurrent reads and writes,
and provide `O(log n)` lookups and inserts. HedgeDB uses a **concurrent skip
list** (`src/db/skiplist/`). While
[this algorithm](https://people.csail.mit.edu/shanir/publications/OPODIS2006-BA.pdf)
is technically not lock-free, it basically behaves as if it were but with
less complexity. The implementation is extracted from
[Folly](https://github.com/facebook/folly/blob/main/folly/ConcurrentSkipList.h).

See the *Failed experiments* section for the path that led to this choice.

## Double-Buffered memtable

The HedgeDB memtable implementation handling cannot be labeled lock-free
because it *will block* under pressure, although there is a widespread usage
of **atomics** for fast synchronization between threads.

The memtable class maintains **two memtable instances**: the active memtable
and the pipelined memtable. Only the active one accepts operations, while
the pipelined one sits there, ready to be used. When the active memtable
gets full, the threads race to acquire the mutex protecting the flush
section:

- The **winner** swaps the consumed memtable with the pipelined one (double
  buffering), and triggers the flush on the background pool.
- The **other threads** briefly spin for a few iterations, waiting for the
  new memtable to become active; after the expiration of the number of
  trials, they will block.
- The pipelined slot is now empty, and one of the background threads
  receives the task of instantiating a new memtable to replace the spot.

This cannot continue indefinitely: eventually compactions will lag behind
flushes (or, less likely, the flushes themselves will lag behind the inbound
traffic). This causes a number of pending flush jobs to pile up.

:::{admonition} Q: What happens if the flush job lags behind the inbound traffic? Does memory grow indefinitely?
:class: tip
**A:** No, there is a hard limit on how many pending flushes are allowed
(config: `max_pending_flushes`). When this limit is reached, new writes are
blocked.
:::

:::{admonition} Q: What if compaction jobs lag behind the flush jobs then?
:class: tip
**A:** You have two choices: accumulating read amplification (because more
and more SSTs are piling up in L0) or **applying backpressure**. HedgeDB by
default applies backpressure if a threshold of SSTs in L0 is exceeded. More
on the backpressure mechanism later.
:::

## Fast Writers and Flusher Synchronization

One of HedgeDB's most sophisticated mechanisms is how it handles memtable
flushes **without blocking concurrent writers**. This is critical for
maintaining high write throughput *and* strict memory safeguards during
flush operations.

### The challenge with multiple Writers

First, let's consider this pseudocode taken from the section of the write
path:

```text
new_size = memtable->size.atomic_add(key_value.size())
if new_size > limit
    flush(memtable)
else
   memtable->insert(key_value)
```

Now, let's consider two threads, T1 and T2 that runs concurrently and the
following situation:

1. `size` = 64; `limit` = 128
2. T1 wants to add a `key_value` with size 64 -> `memtable->size` is now `128`
3. T1 gets preempted by the OS scheduler
4. T2 wants to add a `key_value` with size 32 -> `memtable->size` is now `160`
5. T2 detects that the memtable is full and triggers the flush of `memtable-0` and swap it to `memtable-1`
6. The flusher thread is building the `sst` while iterating the `memtable-0`
7. T1 resumes and update the `memtable-0`
8. T2 writes on `memtable-1`
9. Error! **T1 updates the memtable** while the flusher assumes it has a
   read-only view on it.

-> **DATA LOSS: the flusher has finished and missed the write from T1**

How do we prevent this issue?

HedgeDB solves this problem using a custom synchronization primitive
internally called `rw_sync` which is a declination of the Read-Copy-Update
**Quiescent-State Based Reclamation** (QSBR) with distributed Reference
Counting: `rw_sync`.

What's happening from both actors (Writers and Flusher) perspective is that:

- the Writers temporarily acquire the memtable before updating it by
  increasing the active writers counter;
- A boolean flag indicates whether is possible to acquire the memtable and
  modify it (or, "frozen");
- the Flusher *freezes* the memtable and waits for the writer counter to
  become zero.

```cpp
// include/async/rw_sync.h

struct alignas(64) counter_t {
    std::atomic_int64_t c{0};
};

template <typename T>
class rw_sync {
    T _obj;                              // The protected object (memtable_inner)
    std::atomic_bool _frozen{false};     // Gatekeeper flag
    std::vector<counter_t> _counters;    // Per-thread cache-aligned atomic counters
};
```

At first glance, the vector of counters might look odd. The reason for
using such "distributed reference counting" is that each thread has its own
cache aligned counter (false sharing is prevented), and no performance
degradation due to CPU's
[Cache coherence protocol](https://en.wikipedia.org/wiki/MESI_protocol)
occurs, because the counter's cache line is exclusively updated only from
the Writer thread with the same index.

The wrapped object `_obj` is acquired/released this way:

```text
// acquire
if (rw_sync._frozen)
    return false

rw_sync._counters[this_thread_idx] += 1
return true

// release
rw_sync._counters[this_thread_idx] -= 1
```

For the flusher, checking that the overall reference count is zero is more
expensive though:

```text
rw_sync._frozen = true

ref_count = sum[c in _counters]
while(ref_count != 0)
    ref_count = sum[c in _counters]
    yield
```

Note that after the `rw_sync` is *frozen*, the reference count will get to
zero anytime soon. Either way, this check is performed from the Flusher
that runs on a background thread (or, "non real-time"), so it is totally
acceptable.

## Per-thread Write-Ahead Logs

### Files organization

HedgeDB uses a **per-thread WAL** approach for zero inode contention at the
filesystem level and for write parallelism: NVMe SSDs can easily handle
this.

When the database is initially created, a number of WAL files are generated
with preallocated size (`fallocate` with the `KEEP_SIZE` flag), each of
size `memtable_target_size / num_writer_threads`.

The total file count is `num_writer_threads * (cfg.max_pending_flushes + 2)`
(+1 for the currently active WALs and +1 for the WALs associated with the
pipelined memtable).

WAL files are pooled via `tmc::channel` (internally implemented as a
high-performance async MPMC queue) and reused across flushes: this way we
avoid the repeated overhead of creating, allocating, and deleting files on
the filesystem, and consequently the need to `fsync` the base directory.

### Internal structure and usage

Entries are appended as `{seq_nr, key_size, value_size, key, value, checksum}`
records. On startup, `memtable::replay_wal` reconstructs in-flight writes
from the most recent WAL epoch.

WAL files are intentionally **not opened with `O_DIRECT`**: writing through
the page cache means each insert doesn't require a disk round-trip — the OS
holds the data in RAM and flushes asynchronously. This protects against
application crashes (the OS still owns the data), though not against power
loss. It's an acceptable middle ground for the current prototype stage of
HedgeDB.

A synchronous `pwrite` call is also used instead of going through the
`liburing` event loop. This was experimentally found to be the faster choice.

:::{warning}
**Known gap**: Full crash recovery is not yet implemented. WAL replay works,
but edge cases (partial writes, corrupted files) aren't fully handled.
:::

## Permission Chain for concurrency and temporal consistency

The flush procedure has two degrees of concurrency:

1. Each partition can be flushed concurrently (this is experimentally
   proven to be relevant for performance)
2. Multiple memtable flushes can run in parallel

After the memtable is flushed to SST files, the new SSTs must be added to
the LSM tree **in chronological order**. Inspired by
[this article](https://destel.dev/blog/preserving-order-in-concurrent-go),
HedgeDB uses a **semaphore-based permission chain**:

In `memtable::_flush`:

```cpp
auto next_can_write = std::make_shared<tmc::semaphore>(0);  // Initially locked
auto can_write = std::exchange(this->_can_write, next_can_write);

// Launch flush with permission token
tmc::spawn(this->_flush_inner(..., can_write, next_can_write));
```

In `memtable::_flush_inner`:

```cpp
// ... flush memtable to SSTs ...

// Wait for permission (blocks until previous flush completes)
co_await *can_write;

// ... push SSTs to L0 ...

// Give permission to next flush in chain
next_can_write->release();
```

This creates a **serialized chain** where flush N+1 cannot publish its SSTs
until flush N completes. This ensures SST epoch ordering is preserved.

A similar scheme is used for parallel compaction *within* the same partition.
