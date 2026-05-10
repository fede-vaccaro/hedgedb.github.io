# Failed experiments

*...that didn't make it. What are the lessons learned?*

## The pre-TooManyCooks era: `async::io_executor`

The first thing I worked on in HedgeDB was the `io_uring` event loop,
hosted on a pool of threads. Tasks were submitted through an MPSC queue.

However, at a certain point I started needing coro-based synchronization
primitives that wouldn't block the entire thread, and the executor wasn't
equipped with work stealing.

I even spent some time investigating how to transfer coroutines between
executors, and how to reimplement a small version of `condition_variable`,
taking inspiration from the Linux `futex` implementation.

> *If you're curious: basically, each thread had a queue of tasks that
> could be resumed when a notification was sent to its `io_uring` instance.*

But it got over-complicated and out of scope for this project, and the
performance didn't meet the requirements. Eventually I found `TooManyCooks`,
which was groundbreaking for this project. Don't get me wrong, there are
other great `async` frameworks for C++20, but I thought those were too
heavy as dependencies, and would break the philosophy of this project.

Still, I salute you, `async::io_executor`! A lot of knowledge was acquired
in the process, and I certainly reinvested that into integrating
`TooManyCooks` with HedgeDB.

## Sharded Reference-Counted Clock Cache

You might notice that this experiment is still vaguely present in the
codebase.

Although it now lives in the *future features* list, I actually *did* write
a Clock Cache with lock-free head advancement. But on top of it there was
an rw-lock-protected `tls::sparse_map`, which didn't scale under concurrent
writes.

To mitigate the lock contention, I tried a sharded version that certainly
helped, but still wasn't enough.

It also needs integration with the async framework of the day. I spent a
lot of time integrating it into the `io_executor` mentioned above, with
poor results.

The reason the cache needs to integrate with the async framework is
straightforward: the page you are looking for is reserved in the cache but
the linked buffer hasn't been filled yet. What do you do? The answer is
*"you wait for it to be filled"*. And this leads to a series of interesting
problems that, again, are more in the scope of an async runtime project
than of a KV store.

Maybe I could try again now that I did migrate to `TooManyCooks`.

## The infamous concurrent append-only B-tree

Before settling on Folly's ConcurrentMap as the memtable backend, I tried
multiple solutions. I even ~~vibe-coded~~ wrote a concurrent append-only
B-tree that allowed lock-free insertions within a node.

It was quite fast and I stuck with it for a while. Except that it proved to
be broken and, in some cases, did not maintain internal ordering. Eventually
I decided to fall back to Folly's ConcurrentMap.

I wanted to study the paper and write my own version, but the ConcurrentMap
really does the job and there are other priorities.

### Integrations with K-Way merge

Honestly, this experiment with the k-way merge gave me a really hard time.
One of the tests worth mentioning is that I tried integrating the Clock
Cache because I wanted to push compaction performance by reusing the most
recent SSTs.

This caused enormous complications in the merge procedure, because instead
of reading a full chunk from disk, I first tried to poll the cache for
every 4 KB page in that range, and then fetched the missing pages (or the
entire range) from the filesystem (i.e. "filling the holes").

Lots of time spent here, but I finally scrapped this feature when I
realized that the page cache became a bottleneck once the database size
exceeded the cache size.

Well, it didn't work.

I take it as a big **KEEP IT SIMPLE** lesson learned the hard way.
