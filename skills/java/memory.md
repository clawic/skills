# Memory — OutOfMemoryError, Leaks, and Heap Dumps

The message text after `OutOfMemoryError:` names the region that ran out; each region has a different fix. Getting this wrong means raising `-Xmx` for a problem `-Xmx` cannot touch.

## OOM Taxonomy

| Message | Region | Real cause | First move |
|---|---|---|---|
| `Java heap space` | Heap | Live set exceeds `-Xmx`: a leak, a genuinely bigger workload, or an unbounded cache/queue | Heap dump → dominator tree (below) |
| `GC overhead limit exceeded` | Heap | The JVM spent >98% of recent time in GC recovering <2% of the heap — a leak caught slightly earlier | Same as above; the leak is real, not a tuning issue |
| `Metaspace` / `Compressed class space` | Metaspace | Classes keep being generated or classloaders are never released (redeploys, dynamic proxies, scripting) | Classloader leak hunt (below); `-XX:MaxMetaspaceSize` only makes it fail faster and louder |
| `Direct buffer memory` | Off-heap | `ByteBuffer.allocateDirect` or Netty buffers not released; direct memory is only freed when the owning object is collected | Cap with `-XX:MaxDirectMemorySize`, then find who allocates |
| `unable to create new native thread` | Native | Thread count hit the OS/cgroup limit, or each thread's stack times the count exhausted address space | Count threads in a dump; bound the pool; check `ulimit -u` and cgroup `pids.max` |
| `Requested array size exceeds VM limit` | Heap | An allocation near `Integer.MAX_VALUE` elements — always a computed size bug | Find the computation, not the heap size |
| Killed with exit 137, no OOM in the log | Container | The kernel killed the process: RSS exceeded the cgroup limit while the heap was fine | SKILL.md rule 5 memory budget; `jvm.md` container sizing |

Always run production with `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/app/` — the dump written at the moment of failure is worth more than a week of guessing. Budget disk for it: the file is roughly the size of the live heap.

## Capturing State

```bash
jcmd <pid> GC.heap_info                       # regions, used vs committed
jcmd <pid> GC.class_histogram                 # instances + bytes per class (brief pause)
jcmd <pid> GC.heap_dump /tmp/heap.hprof       # live objects only, full pause: seconds to minutes
jmap -histo:live <pid>                        # forces a full GC first — do not do this casually in prod
jcmd <pid> VM.native_memory summary           # requires -XX:NativeMemoryTracking=summary at startup
```

- A heap dump pauses the application for the whole write; on a 8 GB heap over slow disk that is a real outage. Take it from a drained instance when you can.
- Two dumps 30-60 minutes apart, compared, beat one dump: the leak is what GREW, and any single dump is dominated by legitimately large caches.

## Reading a Heap Dump

1. Open in a dominator-tree tool (Eclipse MAT is the reference). Sort by **retained** size, not shallow size — retained is what freeing the object would actually reclaim.
2. The leak is nearly always ONE root holding a collection. Look at the top 3 retained objects and ask "who keeps this alive": the path-to-GC-root view answers it in one click.
3. Duplicate strings and boxed numbers filling the top of a histogram are a symptom of parse-heavy code, not a leak (`performance.md`).
4. `java.lang.ref.Finalizer` high in the retained list = objects with finalizers queued faster than the finalizer thread drains them; the real fix is removing finalization (`classes.md`).
5. If the top retained object is a classloader, jump to the classloader-leak chain below.

## The Five Leak Shapes

- **Unbounded cache.** A `HashMap` used as a cache with no eviction. Fix: a real cache with a max size and TTL, or `WeakHashMap` only when keys are genuinely identity-scoped (values referencing keys defeat it entirely).
- **Static collection.** `static List<X>` that only ever grows; classic in registries and "recent items" lists. Static means "lives as long as the classloader".
- **Listener/callback never unregistered.** The publisher holds a strong reference to the subscriber forever. Every `addListener` needs a matching `removeListener` on a `finally` or a lifecycle hook.
- **`ThreadLocal` in a pooled thread.** Pool threads never die, so the entry never clears. `remove()` in a `finally` — the entry's *value* is strongly referenced even though the key is weak. Worse with hundreds of virtual-thread-era locals (`concurrency.md`).
- **Classloader leak.** Any static field in a class loaded by a parent classloader that points at an object from the child (a JDBC driver registered in `DriverManager`, a `ThreadLocal` set by app code on a container thread, a shutdown hook) pins the entire child classloader and every class in it. Symptom: Metaspace grows on each redeploy and never recovers.

## Growth That Is Not a Leak

- Heap "grows" up to `-Xmx` by design: an idle JVM does not shrink unless the collector supports uncommitting (G1 does at full GCs; ZGC and Shenandoah uncommit more readily). Judge by the live set AFTER a full GC, not by used heap.
- Direct/native memory grows with connection count in Netty-based stacks — bounded, not leaking, but must be in your budget.
- Metaspace growing for the first minutes of uptime is class loading, which is finished work. Growing forever is a leak.
- The G1 collector's own bookkeeping is roughly 1-2% of heap in region metadata, plus remembered sets — visible in RSS but not in the heap dump.

## Bounding Memory in Code

- Every queue between producers and consumers has a capacity. `new LinkedBlockingQueue<>()` is unbounded — the queue becomes the heap, and the OOM lands on an innocent thread. `Executors.newFixedThreadPool` uses exactly that queue.
- Every batch has a size. `findAll()` in a repository, `Files.readAllLines` on an unknown file, `resultSet` without `setFetchSize` — each one is the same bug: the input decides your memory.
- Streaming beats collecting: `Files.lines` + a pipeline holds one line, `readAllLines` holds the file.
- Caches: pick a max entry count and measure the average retained size per entry once; `maxEntries × avgRetained` must fit a fraction of the heap you can name.

## After the Fix, Prove It

- Run the workload with `-Xlog:gc*:file=gc.log:time,uptime,level,tags` and check the live set after full GCs across the run — flat means fixed, sawtooth with a rising floor means still leaking.
- Set `-Xmx` deliberately low in a soak test: a leak that takes a week at 4 GB takes an hour at 256 MB.
- Add the assertion to the code, not to your memory: bound the collection that leaked and let it throw or evict rather than grow.
