---
hide-toc: true
---

# HedgeDB

```{image} _static/logo_light_mode.png
:alt: HedgeDB
:class: hedge-hero-logo only-light
```

```{image} _static/logo_dark_mode.png
:alt: HedgeDB
:class: hedge-hero-logo only-dark
```

<div class="hedge-hero">

**_Built for the Hardware._**

HedgeDB is a key-value store, built on a partitioned LSM-tree, C++20
coroutines, and `io_uring`. Larger-than-memory, persisted, tuned for modern
NVMe SSDs and modern CPUs.

[Architecture deep-dive »](architecture.md){.hedge-btn .hedge-btn-primary}
[Getting started »](getting-started.md){.hedge-btn}
[GitHub »](https://github.com/fede-vaccaro/HedgeDB){.hedge-btn}
[Discord »](https://discord.gg/kHdxG3AF9x){.hedge-btn}
[![C++20](https://img.shields.io/badge/C%2B%2B-23-blue)](https://github.com/fede-vaccaro/HedgeDB)
[![Status](https://img.shields.io/badge/Version-0.0.1-orange)](https://github.com/fede-vaccaro/HedgeDB)
[![License](https://img.shields.io/badge/license-Apache%20License%202.0-blue)](https://github.com/fede-vaccaro/HedgeDB)
[![Build](https://github.com/fede-vaccaro/HedgeDB/actions/workflows/build.yml/badge.svg)](https://github.com/fede-vaccaro/HedgeDB/actions/workflows/build.yml)

</div>

---

## Features and core design

HedgeDB is an LSM-Tree engine designed to saturate the NVMe device. Inspired by RocksDB, the engine targets write-heavy workloads with uniformly-distributed keys (UUIDs, hashes), and is structured around:

- **Asynchronous execution.** `io_uring` + C++20 coroutines via [TooManyCooks](https://github.com/tzcnt/TooManyCooks),
  a work-stealing scheduler. This allows to perform I/O batching and switching context within user-space.
- **Partitioned LSM-tree.** The key space is sharded into `2^N` independent
  partitions (default 16). Compactions on different partitions run fully in parallel.
- **Size-tiered compaction.** Lower write amplification than leveled, with a
  quotient filter on the read path to skip SSTs that can't contain a key.
- **Per-thread WAL.** Each writer thread owns its own WAL file, no inode
  contention on the hot path.
- **Direct I/O.** reads, flush and compactions use `O_DIRECT`: predictable latencies
  and transparent memory usage, with no IO stalls from [page-cache](direct-io.md) pressure.
- **MVCC.** Snapshot isolation over range scans.

:::{warning}
HedgeDB is currently a prototype and there are a few known gaps. The codebase has a set of test cases, and it's been tested extensively with sanitizers on. It has never run in any production environment, though.
:::

---

## Performance

Performance comparison with RocksDB on a **13th Gen Intel i7-13700H** (6 P-cores + 4 E-cores, 32 GB DDR5 RAM) with a
**Samsung 980 Pro 1TB NVMe**; 100M records, 24-byte keys, 100-byte values:

| Workload | HedgeDB | RocksDB | HedgeDB / RocksDB |
|---|---:|---:|---:|
| Load (100M puts)            | **3.97M ops/s** | 1.14M ops/s | **3.5×** |
| Load + compactions drained  | **3.59M ops/s** | 1.13M ops/s | **3.2×** |
| Read (100M random gets)     | **1.03M ops/s** | 194K ops/s  | **5.3×** |
| Mixed 50/50 read-write      | **1.33M ops/s** | 262K ops/s  | **5.1×** |

---

## Quickstart

Linux only. See [Getting started](getting-started.md) for full prerequisites
and the larger API surface.

### Build

```bash
# install dependencies (liburing is mandatory; hwloc/jemalloc are recommended)
sudo sh install_liburing.sh
sudo apt install libhwloc-dev

# configure & build
cmake . -B build -DCMAKE_BUILD_TYPE=Release -DUSE_JEMALLOC=1 -Wno-dev
cmake --build build -j$(nproc)
```

### Try it out with `benchtool`

The fastest way to confirm everything works end-to-end is to run a small load
through the bundled benchmark CLI:

```bash
# Bump the FD limit (HedgeDB keeps many SST files open)
ulimit -n 1048576

# Load 10M keys with 100-byte values, then read them back
./build/benchtool -m load -n 10000000 -v 100 -p /tmp/hedge_demo
./build/benchtool -m read  -n 10000000 -v 100 -p /tmp/hedge_demo
```

### Hello world from the API

A minimal HedgeDB `Hello World!` example (check `examples/hello_world.cc`):

```cpp
#include <filesystem>
#include <iostream>
#include <string_view>
#include <vector>

#include "db/database.h"
#include "io/static_pool.h"
#include "tmc/sync.hpp"
#include "tmc/task.hpp"

tmc::task<void> run(std::shared_ptr<hedge::db::database> db)
{
    hedge::key_t k{"test_key"};
    std::string v{"Hello, world!"};

    if(auto s = co_await db->put_async(k, std::as_bytes(std::span{v})); !s)
    {
        std::cerr << "put failed: " << s.error().to_string() << "\n";
        co_return;
    }

    auto maybe_value = co_await db->get_async(k);
    if(!maybe_value)
    {
        std::cerr << "get failed: " << maybe_value.error().to_string() << "\n";
        co_return;
    }

    const auto& bytes = maybe_value.value();
    std::string_view readback(reinterpret_cast<const char*>(bytes.data()), bytes.size());
    std::cout << "read back: " << readback << "\n";
}

int main()
{
    const std::filesystem::path db_path = "/tmp/examples_db";
    if(std::filesystem::exists(db_path))
        std::filesystem::remove_all(db_path);

    hedge::io::static_pool::instance()->init(hedge::io::executor_config{
        .name        = "examples-pool",
        .queue_depth = 16,
        .n_threads   = 4,
    });

    auto maybe_db = hedge::db::database::make_new(db_path, hedge::db::db_config{});
    if(!maybe_db)
    {
        std::cerr << "failed to create database: " << maybe_db.error().to_string() << "\n";
        return 1;
    }

    std::shared_ptr<hedge::db::database> db = std::move(maybe_db.value());
    tmc::post_waitable(*hedge::io::static_pool::instance(), run(db)).wait();

    db.reset();
    hedge::io::static_pool::instance()->shutdown();
}
```

---

## What's missing

HedgeDB is a **prototype**. Things that aren't here yet:

- **Full-fledged crash recovery.** WAL replay works, but partial-write and
  corrupted-file edge cases aren't handled.
- **Battle-testing & hardening.** Never run against real-world workloads
  or for long stretches.
- **Cross-platform support.** Linux only (`io_uring`).
- **Block compression.** Many workloads would see real space and
  write-amplification savings from lossless compression.
- **Batched operations.** No batch put/get APIs yet to amortize call overhead.
- **Column families.** One keyspace per database.
- **Large values support.** If `key.size() + value.size()` (a bit less than 4KB) exceeds the index-block
  page size, the flush breaks.

## Future plans

- **Hyper-Clock Cache.** An approximate LRU cache that trades counting precision
  for speed, and plays well with Direct I/O.
- **Key-value separation.** SSTs would store keys plus pointers, with values
  in separate append-only `.vlog` files. This cuts value write-amplification
  during compaction a lot (paired with a GC story).
- **Rate-limiting.** Soft-stall writes when L0 SSTs cross a threshold,
  smoothing the long-tail latencies that come from compaction backlog.
- *Rewrite in Rust?* 👀

---

<!-- Footer template: edit the author line / contact links and bump the date and
     version on each release. -->

Built and maintained by [Federico Vaccaro](https://github.com/fede-vaccaro).
Questions, ideas, or war stories? Open an
[issue](https://github.com/fede-vaccaro/HedgeDB/issues) or reach out via GitHub.

_Last updated: 2026-05-10 · Version: prototype (v0.0.1)_

```{toctree}
:hidden:
:caption: Architecture

architecture
internals/async-model
internals/memtable
internals/sst
internals/compaction
```

```{toctree}
:hidden:
:caption: Posts

direct-io
failed-experiments
```

```{toctree}
:hidden:
:caption: Reference

getting-started
performance
```
