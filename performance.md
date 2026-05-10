# Performance

HedgeDB vs RocksDB on the same machine, same workload, same harness.

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

RocksDB have been tested with Universal Compaction (size-tiered). RocksDB has been provided with `1GB` worth of cache and, `pin_l0_filter_and_index_blocks_in_cache` was enable.

RocksDB was configured in the attempt of matching HedgeDB features. For the specific configurations check `src/benchtool/utils.cc` and `rocksdb/benchtool.cc`.

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
| p99   | 632 µs  | **198 us** |
| p99.9 | 1.05 ms | **295 us** |

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
| Small  (1–100)     | scans/s | **87.5K** | 26.3K   | **3.3×** |
| Small  (1–100)     | keys/s  | **4.38M** | 1.32M   | **3.3×** |
| Medium (512–1024)  | scans/s | **24.9K** | 6.7K    | **3.7×** |
| Medium (512–1024)  | keys/s  | **19.2M** | 5.12M   | **3.7×** |
| Large  (114K–131K) | scans/s | **240**   | 192     | **1.25×** |
| Large  (114K–131K) | keys/s  | **29.5M** | 23.7M   | **1.25×** |

Small and medium scans favor HedgeDB by ~3.3–3.7×. Very large scans
converge — at that range size both engines are bottlenecked by sequential
SSD bandwidth, not the index structure.

## Memory (peak RSS)

| Workload | HedgeDB | RocksDB |
|---|---:|---:|
| Load (100M puts)       | 1.53 GB | 1.03 GB |
| Read (100M gets)       | 455 MB | 1.30 GB |
| Range scans            | 633 MB | 1.30 GB |
| Mixed 50/50 read-write | 1.82 GB | 1.89 GB |

HedgeDB uses more memory during load — the memtable holds pending writes
before they flush to SSTs. On the read path it is significantly lighter:
the SST index cache is demand-filled and shares nothing with the OS page
cache (all reads go through `O_DIRECT`), so memory usage tracks actual
working set rather than page-cache accumulation.

## `io_uring` Queue-depth effect on read latencies

The tests that are shown above, have been executed with the thread-local `io_uring` instance configured with queue-depth `16`.

For very latency-sensitive workloads, the `io_uring` depth queue can be tuned while still maintaining high bandwidth utilization.

Let's see what happens if we reduce the QD to `8` instead:

| Measurement | HedgeDB QD8 | HedgeDB QD16 | RocksDB |
| ---|---:|---:|---:|
| Throughput (reads/s) | 881K | **1.03M** |  193K |
| avg | 108 us |  185 us | **60 us** |
| p50 | 99 us | 155 us | **61 us** |
| p90 | 153.5 us | 298 us | **112 us** |
| p99 | 237.5 us | 632 us | **198 us** |
| p99.9 | 331.5 us | 1025 us | **295 us** |

With this configuration, despite not being able to maximize the device bandwidth (14.5% lower than the peak), we gain substantial improvements on the measured latencies (62.5% decrease). HedgeDB now behaves much closer to RocksDB, proving that it can be adapted even to latency-sensitive scenarios.

:::{admonition} Q: Did you try RocksDB's `MultiGet`? It even support `io_uring`!
:class: tip
**A:** I did try it, but I did not register any meaningful throughput gain, only higher latencies.
:::

## Conclusions

From the results, we can deduce that HedgeDB multi-core and NVMe aware architecture produce the wanted results.

- Writes have **3x** more throughput compared to RocksDB and lower latencies, thanks to the high degree of parallelism, fast synchronization structures and the per-thread WAL.
- Random reads can finally **saturate** the NVMe bandwidth thanks to the `io_uring` integration. However, the maximum throughput comes at the cost of higher latency.
- Short and medium range scan workloads are IOPS-bound, and here the asynchronous architecture shines the most.
- Long range scans are bandwidth-intensive rather than IOPS-intensive, so the concurrent model is less of a differentiator.

## Reproducing

The `benchtool` and `rocksdb_benchtool` binaries that produced these
numbers live in `src/benchtool*` in the repo. See
[Getting started](getting-started.md) for the build steps and CLI flags.
