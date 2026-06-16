# The case for Direct I/O

The choice between Direct I/O vs. Buffered I/O (or, "Page-cache managed") comes with a few trade-offs between them. As stated in the [Manifesto](architecture.md#the-hedge-manifesto), HedgeDB supports Direct I/O and it is enable by default, in order to improve its predictability and being transparent about caching and memory usage.

This does not mean that enabling Direct I/O is the wrong choice, but any engineer that is in the process of configuring a database, should be aware of the effects of the OS Page Cache.

The Linux default is going through the Page Cache for disk I/O operations. Other than sparing the developer annoying alignment conversions, the Page Cache as a double-sided role, depending from the _writer_ or _reader_ perspective.

Both are worth a closer look, so here are some experiments.

_Note: Before each of the experiment that follows, the page cache has been drained with_

```bash
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
```

## Write path

### Random Writes experiment

Writes operation are not immediately directed to the device, but first are buffered until a space threshold is reached (_Dirty pages_) or a timeout expires. This is useful in the attempt of coalescing writes to adjacent offsets.

Let's compare Direct I/O random writes versus Buffered I/O:

- Direct I/O: **545K** IOPS

```bash
$ fio --name=randwrite --rw=randwrite --bs=4k --ioengine=io_uring --iodepth=16 --filesize=20G --filename=bigfile --numjobs=12 --direct=1 --group_reporting --runtime=30s

write: IOPS=545k, BW=2131MiB/s (2234MB/s)(62.4GiB/30001msec); 0 zone resets
lat (usec): min=17, max=7832, avg=351.45, stdev=215.09

```

- Buffered I/O: **432K** IOPS

```bash
$ fio --name=randwrite --rw=randwrite --bs=4k --ioengine=io_uring --iodepth=16 --filesize=20G --filename=bigfile --numjobs=12 --direct=0 --group_reporting --runtime=30s

write: IOPS=432k, BW=1688MiB/s (1770MB/s)(49.5GiB/30002msec); 0 zone resets
lat (usec): min=11, max=711480, avg=443.73, stdev=4045.80
```

Direct I/O is around **26% faster** than the counterpart in terms of throughput; furthermore, the Buffered run shows much **higher tail latency** and standard deviation.

These results can be explained if we look inside on how the whole page-cache machinery works.

We already know that the data is copied from userspace to kernel space (which is already some extra overhead), and that page is marked as dirty in the cache (backed by the the [XArray](https://docs.kernel.org/core-api/xarray.html)).

The dirty pages will be written to disk asynchronously from `kworker` (kernel worker threads). At OS level, there are multiple conditions that might trigger those:

- Every `dirty_writeback_centisecs`, the flushers wakes up polling for work
- If ratio of dirty pages (w.r.t the system memory) exceeds the `dirty_background_ratio`

If the amount of dirty pages exceeds the threshold (`dirty_ratio`), the system **blocks new writes** until the balance is restored.

With `O_DIRECT` none of this happen: no user to kernel space `memcpy`, the page cache update is skipped, no flusher `kworker` is woken up, just straight to the block layer.

### Sequential writes experiment

For simplicity, we will use a single synchronous (`pwrite`) writer, writing one page at a time.

- Direct I/O: **783K** IOPS

```bash
$ fio --name=write --rw=write --bs=4k --ioengine=sync --filesize=20G --filename=bigfile --numjobs=1 --direct=1

write: IOPS=783k, BW=3060MiB/s (3209MB/s)(20.0GiB/6693msec); 0 zone resets
lat (nsec): min=630, max=3447.8k, avg=1158.76, stdev=3908.00
```

- Buffered I/O: **783K** IOPS

```bash
$ fio --name=write --rw=write --bs=4k --ioengine=sync --filesize=20G --filename=bigfile --numjobs=1 --direct=0

write: IOPS=782k, BW=3054MiB/s (3202MB/s)(20.0GiB/6706msec); 0 zone resets
lat (nsec): min=631, max=30195k, avg=1156.43, stdev=17758.98
```

The sequential write benchmark shows almost **identical throughput** between Buffered and Direct I/O.

Latency wise, two different stories are told: once again, the page cache has way more **jitter**: the maximum recorded latency (the outcome is consistent between runs) is almost **9x** higher than the Direct writes run, and the std deviation is **4.5x** higher.

The cause of this jitter is, as the `randwrite` test, due to the writes being stalled until enough dirty pages are drained.

**Does this imply that page cache is obsolete for NVMe?**

Well, no. For general purpose applications still has the effect of _absorbing_ the I/O before landing on the drive, without the need of blocking the entire process.

But for write-intensive systems such as a LSM-tree based database, Direct I/O looks to be the **best choice** for low writes overhead. The negative side is that recent writes won't be hot in cache just after the flush.

These dynamics also apply to HedgeDB (load phase, 100M keys, 24 byte keys, 100 byte payloads):

| Throughput | Direct I/O | Buffered I/O |
| -- | -- | -- |
| 100M writes | 3.97M | 3.36M |
| 100M writes + compaction drain | 3.59M | 2.90M |

This is particularly felt over the compaction backlog drain.

### Read path experiment

#### Random reads

On a read operation, the application (in kernel space) first polls the page cache asking for the pages related to the read coordinates. On cache hit, a `pread` syscall implies basically only a `memcpy` operation, plus the cache, but no actual I/O occurs. The storage device is only interrogated on cache miss.

From task managers like `htop`, we notice now that the memory occupied by the page-cache is not accounted to processes actually operating on that files.
My guess behind this choice is that this cache is managed by the OS rather than the process, and can be evicted at any time if additional room is needed.

This is legitimate, but as database engineers we must consider where the data actually is residing on runtime.

Let me show you a few `fio` runs that clear the matters (I will simplify the output)

- Direct I/O: **813K** IOPS

```bash
$ sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
$ fio --name=randread --rw=randread --bs=4k --ioengine=io_uring --iodepth=16 --filesize=20G --filename=bigfile --numjobs=12 --direct=1 --group_reporting --runtime=30s

read: IOPS=813k, BW=3175MiB/s (3329MB/s)(93.0GiB/30001msec)
lat (usec): min=55, max=3202, avg=235.80, stdev=165.75
```

- Buffered I/O: **3697K** IOPS (!!!)

```bash
$ sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
$ fio --name=randread --rw=randread --bs=4k --ioengine=io_uring --iodepth=16 --filesize=20G --filename=bigfile --numjobs=12 --direct=0 --group_reporting --runtime=30s

read: IOPS=3697k, BW=14.1GiB/s (15.1GB/s)(240GiB/17018msec)
lat (nsec): min=532, max=9375.2k, avg=48188.63, stdev=70338.67
```

- Buffered I/O, but process memory capped to 10GiB (50% of the file size): **1129K** IOPS

```bash
$ sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
$ systemd-run --user --scope -p MemoryMax=10G fio --name=randread --rw=randread --bs=4k --ioengine=io_uring --iodepth=16 --filesize=20G --filename=bigfile --numjobs=8 --direct=0 --group_reporting --runtime=30s

lat (nsec): min=576, max=3294.1k, avg=177017.42, stdev=171716.30
read: IOPS=1129k, BW=4410MiB/s (4625MB/s)(129GiB/30001msec)
```

After running this benchmark, if we hit `free -h` we will also see `10Gi` or more under the "buff/cache` section, representing the test's file.

- Buffered I/O, but process memory capped to 2.5GiB (12.5% of the file size): **721K IOPS**

``` bash
$ sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
$ systemd-run --user --scope -p MemoryMax=2.5G fio --name=randread --rw=randread --bs=4k --ioengine=io_uring --iodepth=16 --filesize=20G --filename=bigfile --numjobs=8 --direct=0 --group_reporting --runtime=30s

read: IOPS=721k, BW=2817MiB/s (2954MB/s)(82.5GiB/30001msec)
lat (nsec): min=576, max=3294.1k, avg=177017.42, stdev=171716.30
```

- Buffered I/O, but process memory capped to 512MiB (2.5% of the file size): **651K IOPS**

``` bash
$ sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
$ systemd-run --user --scope -p MemoryMax=512M fio --name=randread --rw=randread --bs=4k --ioengine=io_uring --iodepth=16 --filesize=20G --filename=bigfile --numjobs=8 --direct=0 --group_reporting --runtime=30s

read: IOPS=651k, BW=2543MiB/s (2666MB/s)(74.5GiB/30001msec)
lat (nsec): min=576, max=3256.8k, avg=196211.53, stdev=145956.83
```

In summary, when rand-reading a 20GB file these are the kind of `randread` performance we might expect.

| Mode | Throughput (IOPS) |
| -- | -- |
| Direct I/O | 813K |
| Buffered (no mem restriction) | 3.69M |
| Buffered (10GB mem restriction) | 1.12M |
| Buffered (2.5GB mem restriction) | 721K |
| Buffered (512 MB mem restriction) | 651K |

With this quick experiment, is now clear that the page cache comes with a overhead that is more and more noticeable as the database grows **larger than the memory** at disposal.

### Which mode does HedgeDB defaults to?

At this point, given the provided experimental evidence, and given the willingness of being [transparent about memory](architecture.md#the-hedge-manifesto) it should be clear now on why HedgeDB defaults on Direct I/O (_with an exception to this rule when dealing with [WAL](internals/memtable.md#details-about-the-write-ahead-log)_).

Also, you will notice that there is some tension between how the Write and the Read path operates.

While buffered writes have some heavy countersides due to the async `kflusher` machinery, there could be many cases where it is still convenient to be backed from the page cache. Plus, it's worth noticing that through `madvise` is possible to instruct the kernel on the needed access pattern.

Finally, the most common solution is offering a custom cache implementation that lives in user space and manages its own eviction policy. That gives the engineer full visibility and control over what stays hot in memory, independent of the OS.