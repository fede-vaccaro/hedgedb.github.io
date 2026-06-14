# The Async Model: `io_uring` + Coroutines

HedgeDB is built around **Linux io_uring** and **C++20 coroutines** via the
brilliant [TooManyCooks](https://github.com/tzcnt/TooManyCooks) work-stealing
scheduler. Every I/O operation is a coroutine that:

1. Submits an SQE (Submission Queue Entry) to the `io_uring` ring
2. Suspends on IO (`co_await`) or blocking operations (yielding the CPU to
   other tasks)
3. Resumes when the CQE (Completion Queue Entry) arrives, by posting the
   task back on the executor.

**No callbacks.** Just sequential-looking code.

Unhappy with the state of the coroutine-based libraries available in C++,
for a long time I stuck to a simple in house-made threadpool, until it
happened that I found `TooManyCooks` while browsing Reddit (see the *Failed
experiments* section).

## Executor Topology

Each thread in a pool owns an `io_uring` ring instance. This let `io_uring` to exploit some optimization that work under these circumstances (e.g. `IORING_SETUP_SINGLE_ISSUER`).

HedgeDB instantiates two threadpools:

- **Foreground executor**: Thread pool where the DB primary operations (`put`,`get`, `remove`, `scan`) are processed.
- **Background executor**: Thread pool handling the memtable flush-to-SST
  and the compaction jobs.

### CPU-awareness in HedgeDB

Despite it is optional, it is heavily recommended to build HedgeDB with **`hwloc`**. This is a feature directly inherited from `TooManyCooks`, but it allows to exploit the CPU-specific hardware architecture.  

Modern processors have **multiple** and/or **heterogeneous core groups**. Recent Intel architectures present *Performance Cores* and *Efficient cores*, grouped in different number and with separated cache layouts;
AMD Ryzen processors have multiple *CCXs*; this implies that sharing any data between different group is potentially detrimental, and that different core type have different performance capability.  

`hwloc` allows the software to inspect the CPU core groups and set the FG and BG **thread affinity** accordingly.  
For instance, on a hybrid-core Intel CPU, we found experimentally gained significant performance improvement on `insertion` workload (around +25%) when executing the Foreground pool run on the P-cores group, and the Background pool on the E-cores groups.

## Why the async model is necessary

> **Queue depth** (QD): In NVMe, queue depth is the number of I/O commands that can be outstanding (submitted but not yet completed) in a single submission queue at one time. NVMe SSDs often saturate at QD32 or QD64.

Let's take RocksDB as example. The foreground operations are synchronous:
under the hood it uses synchronous `pread` and `pwrite` syscalls. This means
that a CPU without a large number of threads won't saturate an NVMe IOPS capacity,
since every `pread`/`pwrite` runs at Queue Depth 1. Over-spawning threads w.r.t to the CPU
core count improves throughput only to limited extent, because we will end up
paying the cost of expensive **context-switches**. Memory-mapped storage
isn't a great solution either: while writes to memory-mapped regions are mostly asynchronous, reads cannot: the thread will be blocked on page swap
anyway. In both cases we are subject to the OS implementation which don't
really play in our favor. Here a great
[explanation](https://youtu.be/1BRGU_AS25c?si=wzDmVb1Ki1_H1toF) on the
reasons why.

### Prior art

Previous efforts in the same direction of coroutine-based database are most notably
[ScyllaDB](https://github.com/scylladb/scylladb) built on the [Seastar framework](https://github.com/scylladb/seastar), and CockroachDB's [Pebble](https://github.com/cockroachdb/pebble); Pebble inherits the Go language asynchronous design, but heavily relies on the OS page cache.

## I/O Batching

As mentioned, NVMes present the highest throughput (*nb. it costs higher latency*) at high _Queue Depth_ (e.g. 512 outstanding requests),
i.e. the number of requests that can be submitted concurrently in order to exploit the internal parallelism
of the device. HedgeDB's concurrency model based on coroutines allows to integrate
batching quite seamlessly. I/O requests filed from the same thread can be batched
while still writing code that is quite close to the sequential/blocking
version.

If you look at the codebase, you will notice many calls like

```cpp
int res = co_await hedge::io::read(fd, data, size, offset);
```

This looks almost identical to the synchronous counterpart, yet it enables
multiple I/O requests to be filed at once from one single thread.

**Under the hood**

HedgeDB implements a custom awaitable type that integrates `io_uring` into `TooManyCooks`. When such objects are `co_await`ed, an I/O request is filed to the thread local `io_ctx`, each holding a `io_uring` instance. With the help of the awaitable object, the `io_ctx` *suspends* the execution and stores  internally the I/O request type and coordinates the coroutine handle for resuming it later (or the coroutine *yields*). Now, another task is picked for execution from the TMC executor work-queue, that might potentially file another I/O request. This process continues until one of these conditions fires:

1. There are enough pending requests to fill the `io_uring` submission queue;
2. There is no available task to be executed for the current executor.



The `io_ctx` now calls `io_uring_submit_and_wait` on the requests batch. As
the name suggests, this call submits a batch of I/O requests (or, Submission
Queue Entries) to the system *and* synchronously waits for at least a Completion Queue
Entry to land.  

Now that the I/O requests are fulfilled, every suspended task gets resumed by being scheduled on its reference threadpool.
