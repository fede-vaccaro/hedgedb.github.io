# Getting started

HedgeDB is **Linux-only** (it heavily leverages `io_uring`) and currently a
prototype: it has not been extensively tested, and the code can't be
considered production-ready.

## Dependencies

- Linux kernel `6.0` or higher
- `gcc 13` or higher
- [`liburing`](https://github.com/axboe/liburing) `2.14` (you can run
  `install_liburing.sh`)
- Other dependencies are managed via CMake

*Optional but highly recommended:*

- [`hwloc`](https://www.open-mpi.org/projects/hwloc/) for CPU-aware thread localization (`sudo apt install hwloc libhwloc-dev`)
- A concurrency-friendly memory allocator like
  [`jemalloc`](https://github.com/jemalloc/jemalloc) or
  [`tcmalloc`](https://github.com/google/tcmalloc)

## Build

The full list of dependencies for running tests can be found `.github/workflows/tests.yml`.
A few dependencies are managed from CMake.

```bash
# install liburing (mandatory)
sudo sh install_liburing.sh

# install jemalloc (optional)
sudo sh install_jemalloc.sh

# install hwloc (optional, highly recommended)
sudo apt install libhwloc-dev

# Configure
cmake . -B build -DCMAKE_BUILD_TYPE=Release -DUSE_JEMALLOC=1 -Wno-dev

# Build
cmake --build build -j$(nproc)

# Hello World
./build/hello_world
```

## Build with tests

```bash
# Increase file descriptors limit
ulimit -n 1048576

# Build tests
cmake -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=1 -Wno-dev 
cmake --build build -j$(nproc)

# Run tests 
ctest --test-dir build -V
```

## Run benchmarks

`benchtool` is the unified benchmark CLI. Build it with the regular cmake
build, then:

```bash
# Increase file descriptor limit, needed for managing many SST files
ulimit -n 1048576

# Write 50M keys with 100-byte values
./build/benchtool -m write -n 50000000 -v 100 -p /tmp/bench_db

# Read them back (loads existing db at path)
./build/benchtool -m read  -n 50000000 -v 100 -p /tmp/bench_db

# Mixed 50/50 workload (preload the database first)
./build/benchtool -m rw    -n 50000000 -v 100 -p /tmp/bench_db

# Range scans (small / medium / large tiers)
./build/benchtool -m range -n 10000    -v 100 -p /tmp/bench_db
```

| Flag | Long form | Default | Description |
|------|-----------|---------|-------------|
| `-n` | `--num_ops` | `1000000` | number of operations |
| `-v` | `--vsize` | `100` | value size in bytes |
| `-m` | `--mode` | `write` | `write` \| `read` \| `rw` \| `range` |
| `-p` | `--path` | `/tmp/bench_db` | database directory |

`read`, `rw`, and `range` modes load an existing database from `--path`; if
none is found they exit with an error.

## API examples

> ### A note on the integration
>
> The main challenge with such execution model in C++ is that, compared to other languages there is no general consensus from the community for a standard asynchronous I/O model (like [tokio](https://github.com/tokio-rs/tokio) in Rust), and for sure there is no standardized approach (like the Go concurrency model itself).  
> I understand that this creates friction for embedding HedgeDB into existing systems, hence the current setting constitutes the bedrock on which building applications rather then the opposite.  
> However, it is worth noticing that `TooManyCooks` offer some degree of flexibility that might be worth exploring: the [integration with asio](https://github.com/tzcnt/tmc-asio) example is a materialization of such.

The public surface lives in `db/database.h` and `io/static_pool.h`. Async
APIs return `tmc::task<...>` values that must be `co_await`-ed inside a
coroutine; to drive them from synchronous code, post the entry-point task
onto the worker pool with `tmc::post_waitable` and `wait()` on the future.

A complete runnable example lives in
[`examples/hello_world.cc`](https://github.com/fede-vaccaro/HedgeDB/blob/main/examples/hello_world.cc); the snippets below
are extracted from it. For throughput-oriented patterns (multi-threaded
workers, bounded in-flight ops), see `benchtool/`.

### Initialize the pool and open the database

```cpp
hedge::io::static_pool::instance()->init(hedge::io::executor_config{
    .name = "examples-pool",
    .queue_depth = 16,
    .n_threads = 4,
});

auto maybe_db = hedge::db::database::make_new(db_path, hedge::db::db_config{});
if(!maybe_db)
{
    std::cerr << "failed to create database: " << maybe_db.error().to_string() << "\n";
    return 1;
}

std::shared_ptr<hedge::db::database> db = std::move(maybe_db.value());
```

Use `database::load(path, cfg)` to reopen an existing database. See
`benchtool/utils.cc` for a `db_config` filled with non-default values
(`memtable_budget_bytes`, `num_partition_exponent`, `use_direct_io`, …).

### Put and get

Inside a `tmc::task<void>`:

```cpp
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
```

### Range scan

`database::scan` is synchronous to construct (it snapshots the SST set
under the partition lock); only `iterator.next()` is awaitable. Pass
`std::nullopt` for an open upper bound.

```cpp
hedge::key_t range_key_start{...};
hedge::key_t range_key_end{...};

auto maybe_it = db->scan(range_key_start, range_key_end);
if(!maybe_it)
{
    std::cerr << "failed to create iterator: " << maybe_it.error().to_string() << "\n";
    co_return;
}

auto it = std::move(maybe_it.value());
while(auto entry = co_await it.next())
{
    auto& [key, value] = entry.value();
    // process key, value
}
```

A scan iterator is bound to a single partition (selected from the lower
bound). To sweep the whole key space, issue one scan per partition; see
`benchtool/range.cc` for a worker that randomly samples lower bounds.

### Bridge sync code into the async world

Wrap the entry point as a coroutine, post it onto the static pool, and
`wait()` on the returned future:

```cpp
tmc::task<void> task do_something(auto db)
{
    ...
}

tmc::post_waitable(*hedge::io::static_pool::instance(), do_something(db)).wait();
```

### Shutdown

```cpp
db.reset();
hedge::io::static_pool::instance()->shutdown();
```

## Where to look next

| Want to... | Start here |
|------------|------------|
| Understand the API | `src/db/database.h` |
| Modify memtable | `src/db/memtable.{h,cc}` |
| Change SST format | `src/db/sst.{h,cc}`, `src/db/block.{h,cc}` |
| Tweak compaction | `src/db/sst_manager.cc` |
| Write tests | `test/*.spec.cc` |

## What's missing

If it wasn't clear enough already, HedgeDB is a **prototype**. Here's what
isn't there yet:

- **Full-fledged crash recovery**: WAL replay works, but edge cases
  (partial writes, corrupted files) aren't handled.
- **Battle-testing & hardening**: never tested in the wild with real-world
  workloads or long execution periods. Some edge cases are not handled.
- **Cross-platform support**: it's Linux-only (`io_uring` dependency).
- **Block compression**: many workloads can get meaningful size reduction
  from lossless compression algorithms, leading to noticeably lower space
  and write amplification.
- **Batched operations**: batched writes and reads to amortize overheads.
- **Column family support**: no explicit column family support.
- **Large values support**: if `key.size() + value.size()` exceeds the
  index block page size, compaction will break.

**Possible future features**:

- **Hyper-Clock Cache**: an approximate LRU cache that trades
  "least-recently-referenced" counting precision for a simpler and faster
  algorithm.
- **Key-value separation**: SSTs would store only keys and pointers, with
  values in separate append-only `.vlog` files, dramatically reducing
  value write amplification during compaction (also depends on the GC
  implementation).
- Improved handling of non-uniform key distributions (or implementing
  trivial moves).
- **Rate-limiting**: stall writes at a specific rate if the SSTs in L0
  exceed a soft threshold; this smooths long-tail latencies.
