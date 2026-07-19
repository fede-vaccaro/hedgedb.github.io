# Performance analysis

In this article, is presented a quick study around **HedgeDB performance** in terms of operation throughput and latency on different real-world workloads.

To provide a well-known reference to compare against, RocksDB has been chosen because it's embeddable and LSM-tree based aswell.

RocksDB has been configured with Universal Compaction (Size Tiered) and other, and the same data is submitted. For further details, check the [rocksdb/benchtool.cc script](https://github.com/fede-vaccaro/HedgeDB/blob/main/rocksdb/benchtool.cc) on GitHub.

## Setup

| | |
|---|---|
| **CPU**     | 13th Gen Intel i7-13700H (14 cores / 20 threads) |
| **RAM**     | 32 GB DDR5|
| **Storage** | Samsung 980 Pro 1TB NVMe |
| **Records** | 100M, 24-byte keys, 100-byte values (~12 GB raw) |
| **Key space**    | uniformly-distributed random |

Both RocksDB and HedgeDB have been tested with `O_DIRECT` I/O mode, with 12 threads plus 8 background threads (for flush and compaction), reflecting the test CPU architecture (6 P-cores with SMT and 4+4 E-cores).

In the HedgeDB benchmarks, the operations are submitted through the `TooManyCooks` coroutine-based threadpool; in the RocksDB the operations are submitted just via `std::thread`.

RocksDB has been tested with Universal Compaction (size-tiered). RocksDB has been provided with `1GB` worth of cache and `pin_l0_filter_and_index_blocks_in_cache` was enabled.

RocksDB was configured in an attempt to match HedgeDB's features. For the specific configurations check `src/benchtool/utils.cc` and `rocksdb/benchtool.cc`.

## Throughput

| Workload | HedgeDB | RocksDB | HedgeDB / RocksDB |
|---|---:|---:|---:|
| Load (100M puts)            | **3.97M ops/s** | 1.14M ops/s | **3.5×** |
| Load + compactions drained  | **3.59M ops/s** | 1.13M ops/s | **3.2×** |
| Read (100M random gets)     | **1.03M ops/s** | 194K ops/s  | **5.3×** |
| Mixed 50/50 read-write      | **1.33M ops/s** | 262K ops/s  | **5.1×** |

## Latency

### Read (read-only workload)

HedgeDB's per-request latency is higher than RocksDB's despite its 5.3× throughput advantage. This is the expected tradeoff of the batching model: each thread runs its own `io_uring` ring at QD16, keeping multiple I/O requests in flight simultaneously. More requests in flight means higher aggregate throughput, but each individual request spends more time waiting in the queue. See the [Queue-depth effect](#io_uring-queue-depth-effect-on-read-latencies) section below for a direct QD8 vs QD16 comparison.

| Percentile | HedgeDB | RocksDB |
|---|---:|---:|
| avg   | 185 µs  | **60 µs**  |
| p50   | 155 µs  | **61 µs**  |
| p90   | 298 µs  | **112 µs** |
| p99   | 632 µs  | **198 µs** |
| p99.9 | 1.05 ms | **295 µs** |

### Write (memtable insert+WAL append)

| Percentile | HedgeDB | RocksDB |
|---|---:|---:|
| avg   | **2.73 µs** | 10.28 µs |
| p50   | **2.0 µs**  | 9.5 µs   |
| p99   | **6.0 µs**  | 17.0 µs  |
| p99.9 | **23.5 µs** | 25.5 µs  |

### Read latency under the mixed workload

| Percentile | HedgeDB | RocksDB |
|---|---:|---:|
| avg | 285 µs  | **84 µs**  |
| p50 | 237 µs  | **72 µs**  |
| p90 | 430 µs  | **136 µs** |
| p99 | 1.09 ms | **281 µs** |

## Range scans

| Range size | Metric | HedgeDB | RocksDB | HedgeDB / RocksDB |
|---|---|---:|---:|---:|
| Small  (1-100)     | scans/s | **87.5K** | 26.3K   | **3.3×** |
| Small  (1-100)     | keys/s  | **4.38M** | 1.32M   | **3.3×** |
| Medium (512-1024)  | scans/s | **24.9K** | 6.7K    | **3.7×** |
| Medium (512-1024)  | keys/s  | **19.2M** | 5.12M   | **3.7×** |
| Large  (114K-131K) | scans/s | **240**   | 192     | **1.25×** |
| Large  (114K-131K) | keys/s  | **29.5M** | 23.7M   | **1.25×** |

Small and medium scans favor HedgeDB by ~3.3-3.7×. Very large scans
converge: at that range size both engines are bottlenecked by sequential
SSD bandwidth, not the index structure.

## Memory (peak RSS)

| Workload | HedgeDB | RocksDB |
|---|---:|---:|
| Load (100M puts)       | 1.53 GB | 1.03 GB |
| Read (100M gets)       | 455 MB | 1.30 GB |
| Range scans            | 633 MB | 1.30 GB |
| Mixed 50/50 read-write | 1.82 GB | 1.89 GB |

HedgeDB uses more memory during load, since the memtable holds pending writes
before they flush to SSTs. On the read path it is significantly lighter:
the SST index cache is demand-filled and shares nothing with the OS page
cache (all reads go through `O_DIRECT`), so memory usage tracks actual
working set rather than page-cache accumulation.

## `io_uring` Queue-depth effect on read latencies

The tests shown above were executed with the thread-local `io_uring` instance configured with queue-depth `16`.

For very latency-sensitive workloads, the `io_uring` queue depth can be tuned while still maintaining high bandwidth utilization.

Let's see what happens if we reduce the QD to `8` instead:

| Measurement | HedgeDB QD8 | HedgeDB QD16 | RocksDB |
| ---|---:|---:|---:|
| Throughput (reads/s) | 881K | **1.03M** |  193K |
| avg | 108 µs |  185 µs | **60 µs** |
| p50 | 99 µs | 155 µs | **61 µs** |
| p90 | 153.5 µs | 298 µs | **112 µs** |
| p99 | 237.5 µs | 632 µs | **198 µs** |
| p99.9 | 331.5 µs | 1025 µs | **295 µs** |

With this configuration, despite not being able to maximize the device bandwidth (14.5% lower than the peak), we gain substantial improvements on the measured latencies (62.5% decrease). HedgeDB now behaves much closer to RocksDB, proving that it can be adapted even to latency-sensitive scenarios.

:::{admonition} Q: Did you try RocksDB's `MultiGet`? It even supports `io_uring`!
:class: tip
**A:** I did try it, but I did not register any meaningful throughput gain, only higher latencies.
:::

## Performance update on Zen 5 + Gen5 NVMe

Recently, I upgraded my workstation to some latest-generation hardware: the build has 
a **9900X3D Zen5 CPU** equipped with 12 cores (24 threads with SMT), 32 GB of DDR5 RAM 
and a Gen5 NVMe: a **Crucial T710 2TB**! I chose this model because it's
better at sustained writes ([source](https://www.techpowerup.com/review/crucial-t710-2-tb/6.html))
compared to other alternatives. 

I personally tested it against a Samsung 9100 Pro 2TB and can confirm those results hold
for HedgeDB too: the Samsung really fell over on a benchmark with larger 
values (512 bytes). During this large-value test, the write amplification caused by compaction was noticeably high. That put the entire burden on the NVMe, which delivered very poor write throughput once its SLC cache filled up, so I decided to get the Crucial instead.

The test setup is almost the same as before (HedgeDB received some updates and small improvements, though). 
Once again the workload was adapted to the CPU architecture: the foreground tasks (writes, reads and range scans)
run on a threadpool pinned to the first CCX; background tasks (flush and compactions) run on the second CCX.

A small note: I'm aware that in a real-world use case, server CPUs are different from desktop ones: they usually have more cores running at a lower frequency (Zen5 runs at ~5GHz!). Industrial NVMe SSDs are
also built differently from consumer hardware, since they are made to perform consistently over sustained 
workloads, while my benchmarks last for no longer than tens of seconds.

With these caveats in mind, it is still quite interesting to explore how **HedgeDB scales** when provided with more power.

Let's look at the results.

### Throughput

| Workload | HedgeDB | RocksDB | HedgeDB / RocksDB | HedgeDB (laptop) | RocksDB (laptop) |
|---|---:|---:|---:|---:|---:|
| Load (100M puts)            | **8.07M ops/s** | 1.34M ops/s | **6.0×** | 3.97M ops/s | 1.14M ops/s |
| Load + compactions drained  | **7.94M ops/s** | 1.33M ops/s | **5.9×** | 3.59M ops/s | 1.13M ops/s |
| Read (100M random gets)     | **2.23M ops/s** | 296K ops/s  | **7.5×** | 1.03M ops/s | 194K ops/s  |
| Mixed 50/50 read-write      | **2.57M ops/s** | 423K ops/s  | **6.1×** | 1.33M ops/s | 262K ops/s  |

From the table above, you can see how both HedgeDB and RocksDB behave on my desktop workstation against 
the laptop I ran the first comparison on.

It's clear that HedgeDB, thanks to its parallel design, is capable of crunching **even more write operations** and widening
the gap with RocksDB.

I should point out that, judging from internal tests, the bottleneck here seems to be
the **Concurrent SkipList** rather than the storage device itself, hinting that the device saturation point might
not be reached yet.

When running RocksDB, it is easy to see that the write workload is **software-constrained**, since it does not show 
the **same gains** from the new hardware that HedgeDB does.

About the `Read 100M` tests, HedgeDB still saturates the drive at **2.2M random lookups/s**, 
the T710's nominal random-read IOPS. The story here is still the same as before: batched I/O operations 
(submitted via `io_uring`) are **mandatory** for saturating the SSD controller submission queue (more on this in
[async model](internals/async-model.md)).

### Latency

#### Read (point-lookup workload)

| Percentile | HedgeDB | RocksDB |
|---|---:|---:|
| avg   | 172 µs | **40 µs**  |
| p50   | 155 µs | **48 µs**  |
| p90   | 259 µs | **59 µs**  |
| p99   | 414 µs | **105 µs** |
| p99.9 | 578 µs | **142 µs** |

The same tradeoff as before. HedgeDB keeps 32 I/O requests in flight per thread (the earlier test ran at QD16),
so each individual read waits longer while the device stays busy.

#### Write (load phase)

| Percentile | HedgeDB | RocksDB |
|---|---:|---:|
| avg   | **1.48 µs** | 8.83 µs  |
| p50   | **1.31 µs** | 8.50 µs  |
| p99   | **5.12 µs** | 12.00 µs |
| p99.9 | **7.95 µs** | 14.50 µs |


### Range scans

| Range size | Metric | HedgeDB | RocksDB | HedgeDB / RocksDB |
|---|---|---:|---:|---:|
| Small  (1-100)     | scans/s | **134K**  | 42.7K   | **3.1×** |
| Small  (1-100)     | keys/s  | **6.68M** | 2.14M   | **3.1×** |
| Medium (512-1024)  | scans/s | **39.6K** | 14.5K   | **2.7×** |
| Medium (512-1024)  | keys/s  | **30.4M** | 11.1M   | **2.7×** |
| Large  (114K-131K) | scans/s | **431**   | 396     | **1.09×** |
| Large  (114K-131K) | keys/s  | **53.1M** | 48.7M   | **1.09×** |

The shape is the same as on the laptop: a clear win on short and medium
scans, near-parity on the large ones where both engines just stream from the
device.

### Memory (peak RSS)

| Workload | HedgeDB | RocksDB |
|---|---:|---:|
| Load (100M puts)       | 1.14 GB | 1.00 GB |
| Read (100M gets)       | 475 MB  | 1.26 GB |
| Range scans            | 910 MB  | 1.31 GB |
| Mixed 50/50 read-write | 1.78 GB | 2.01 GB |

Note that RocksDB always had HyperClockCache enabled, with 1 GiB allocated 
for caching filters and index blocks.

## Conclusions

From the results, we can deduce that HedgeDB's multi-core, NVMe-aware architecture delivers what it was designed for.

- HedgeDB can accommodate a wider flow of requests, showing a **higher throughput** compared to RocksDB (and lower latencies), thanks to the high degree of parallelism, fast synchronization structures and the per-thread WAL.
- Random reads can **finally saturate** the NVMe bandwidth thanks to the `io_uring` integration. Although maximum throughput comes at the cost of higher latency, the user can tune the submission queue depth based on the needs of the use-case.
- Short and medium range scan workloads **are IOPS-bound**, and here the HedgeDB's I/O batching shines the most.
- Long range scans are bandwidth-intensive rather than IOPS-intensive, so the concurrent model based on `io_uring` model is less of a differentiator.

## Reproducing

The `benchtool` and `rocksdb_benchtool` binaries that produced these
numbers live in `src/benchtool*` in the repo. See
[Getting started](getting-started.md) for the build steps and CLI flags.
