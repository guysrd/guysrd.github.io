---
layout: default
---

# The Epoll UAF

In early 2026, a quiet one-line patch landed in the Linux kernel. [Commit 07712db80857](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=07712db80857d5d09ae08f3df85a708ecfc3b61f), by Nicholas Carlini, changed a `kfree()` to `kfree_rcu()` in `fs/eventpoll.c`. The commit message: "eventpoll: defer struct eventpoll free to RCU grace period." No further context.

That one call was covering a use-after-free that had been reachable from any unprivileged process for over a year, on every Linux system running a 6.x kernel with the affected optimization backported. Every Android phone. Every GKI-based device.

How does the bug work? How hard is it to trigger? And what can you actually do with it on a modern hardened kernel? This post is my attempt to answer those questions. I spent several weeks on a Pixel 10 (Frankel, kernel 6.6.102) pulling this apart, and what I found was less about the bug itself and more about the optimization that introduced it -- and why the people who wrote that optimization had no reason to think they'd broken anything.

---

## Some Context on epoll

If you've run a Linux server you've used epoll indirectly. It's the kernel's scalable I/O notification mechanism -- the thing that lets nginx watch tens of thousands of sockets without blocking a thread per connection. Three syscalls: `epoll_create()` makes an instance, `epoll_ctl()` adds or removes file descriptors to watch, `epoll_wait()` blocks until something happens.

The interesting twist is that an epoll fd is itself a file descriptor. You can add an epoll instance to another epoll instance. This creates a directed graph of watchers watching watchers, and to prevent cycles or unbounded nesting, the kernel runs validation code inside `epoll_ctl(ADD)` that walks the graph checking for loops and depth violations.

That graph-walking code is where the bug lives.

---

## The Two Structures

Two kernel objects matter here.

![epoll data structures and the UAF](1.svg)

**`struct eventpoll`** is per-instance state. One per `epoll_create()` call. It has the wait queue, the red-black tree of watched items, and a field called `refs` at offset 176 -- an hlist head linking every `epitem` that points back at this instance from somewhere else. Think of it as the incoming-edges list in the graph.

**`struct epitem`** is one per (epoll instance, watched fd) pair. It has `epi->ep`, a pointer to its owning `eventpoll`. If the watched fd is another epoll instance, the `epitem` is also linked into that target's `refs` hlist through `fllink`.

The graph walker iterates `ep->refs`. For each `epitem`, it follows `epi->ep` to reach the parent `eventpoll`. Then it recurses.

That dereference -- `epi->ep` -- is the use-after-free.

---

## The 2023 Optimization

Before March 2023, every `epoll_ctl(ADD)` involving the nested case acquired a global mutex called `epmutex`. Under HTTP benchmarks, 58% of CPU time was spent contending on it. A brutal bottleneck.

The optimization patch replaced the global mutex with a per-instance `refcount_t`, added a `dying` flag to `struct epitem`, and narrowed the remaining lock (`epnested_mutex`) to only be held during actual graph walks. The result: a 60% throughput improvement. A real, well-measured win.

The authors carefully audited the race between two specific close paths: `ep_clear_and_put()` (closing an epoll fd) and `eventpoll_release_file()` (closing a watched fd). That race is correctly handled by the new refcount + dying flag.

But there was a third path. The graph walkers -- `ep_get_upwards_depth_proc` and `reverse_path_check_proc` -- were running under `rcu_read_lock()` and iterating `ep->refs` while other threads could be tearing down the `eventpoll` structures they were pointing at. The old global mutex had been incidentally serializing this. Nobody noticed because the walkers don't touch any of the data the mutex was nominally protecting. They just follow pointers through it.

---

## The Bug

Here's the function. It's the "upward" walker, called during `epoll_ctl(ADD)`:

```c
static int ep_get_upwards_depth_proc(struct eventpoll *ep, int depth)
{
    int result = 0;
    struct epitem *epi;

    if (ep->gen == loop_check_gen)
        return ep->loop_check_depth;

    hlist_for_each_entry_rcu(epi, &ep->refs, fllink)
        result = max(result, ep_get_upwards_depth_proc(epi->ep, depth + 1) + 1);
                                                       /* ^^^^^^^ */
    ep->gen = loop_check_gen;
    ep->loop_check_depth = result;
    return result;
}
```

The traversal runs under `rcu_read_lock()`. Each `epitem` is safe to access -- when unlinked, it's freed via `call_rcu()`, so RCU keeps it alive for the duration of any read-side critical section. There's even a comment in the source acknowledging the RCU reader:

```c
/* The rcu read side, reverse_path_check_proc(), does not make
 * use of the rbn field.
 */
call_rcu(&epi->rcu, epi_rcu_free);
```

That comment is correct about the `epitem`. It says nothing about what the `epitem` points to.

The teardown when an epoll fd is closed:

```c
static void ep_free(struct eventpoll *ep)
{
    mutex_destroy(&ep->mtx);
    free_uid(ep->user);
    wakeup_source_unregister(ep->ws);
    kfree(ep);
}
```

`kfree()`. Not `kfree_rcu()`. The `eventpoll` is freed immediately, with no deference to any RCU grace period.

A pointer being safe to *read* is not the same as what it points to being safe to *use*. The walker holds `epi` alive under RCU (correct), reads `epi->ep` (just a pointer load, fine), then dereferences the target (which may have been freed and reallocated as something else entirely). That's the use-after-free.

---

## Triggering the Race

![The race timeline](2.svg)

I initially tried the obvious approach: two threads on different CPUs, one walking the graph, one closing an epoll fd. This doesn't work. The window between loading `epi` from the hlist and dereferencing `epi->ep` is a handful of ARM64 instructions. Cache coherence latency across physical cores is wider than the gap. The cross-CPU race is basically unwinnable.

What does work is same-CPU preemption. The Pixel 10 runs `CONFIG_PREEMPT=y` and `CONFIG_PREEMPT_RCU=y`, which means `rcu_read_lock()` does not disable preemption -- it just bumps a per-task counter. A timer tick during the graph walk can yield the CPU to the closer thread, even while the walker is mid-RCU.

Some numbers from the device (`CONFIG_HZ=250`, so a tick every 4 ms):

- 4,096 epoll parents: the walk takes ~400 us. Too short, rarely hits a tick boundary.
- 8,000 parents: ~2 ms. Overlaps a tick often enough. **Hit rate about 4% per attempt.**

I also stumbled on a scheduler subtlety that took me an embarrassingly long time to figure out. CFS prefers threads with low virtual runtime. If the closer thread busy-waits for the signal, it accumulates vruntime and CFS *deprioritizes* it -- opposite of what you want. The fix, which is counterintuitive: have the closer `usleep(1000)` in a loop while waiting. When it wakes, its vruntime is near zero, CFS strongly prefers it, and it preempts the walker. I wasted at least a day on this before checking the vruntime values.

One more thing: the CPU frequency governor matters. The Pixel's default `sched_pixel` governor was throttling CPU 0 to 729 MHz at idle, which stretched the traversal timing enough to kill the race entirely. Setting `performance` mode (2,246 MHz) was necessary. This is the kind of detail that doesn't show up in a theoretical analysis but burns you on a real device.

---

## What Gets Written Where

When the walker reaches a freed `eventpoll`, it touches three offsets of `struct eventpoll`:

| offset | field              | size | action          |
|--------|--------------------|------|-----------------|
| 168    | `gen`              | u64  | read, then write `loop_check_gen` |
| 176    | `refs.first`       | ptr  | read as hlist pointer |
| 184    | `loop_check_depth` | u8   | write `result` (typically 0) |

`struct eventpoll` is 200 bytes, allocated from `kmalloc-256` (order-1 slabs, 32 objects per slab, `cpu_partial=52` on this device). After `kfree()`, the slot goes back to SLUB's per-CPU freelist. With `init_on_free=1` (set via kernel cmdline on this device, not the config default), the 256-byte slot is zeroed on free.

Everything hinges on what occupies offset 176 when the walker gets there:

- **Zero** (from `init_on_free`): the hlist looks empty. The walker skips the recursion, writes `loop_check_gen` at offset 168 and a zero byte at offset 184, and returns. A silent 9-byte corruption of whatever `kmalloc-256` object now lives in that slot.

- **Nonzero**: the walker follows offset 176 as a pointer to an `epitem`, computes `container_of()`, dereferences `epi->ep`, and recurses into arbitrary kernel memory. On garbage data, the kernel panics immediately.

The interesting case is a controlled nonzero value. If you can reclaim the freed slot with an object where you choose what's at offset 176, you can steer the walker's recursion. Each level of recursion writes `loop_check_gen` (a u64 global counter) and a zero byte at fixed offsets from the target. Chain a few levels and you have a constrained kernel write primitive.

---

## Cross-Cache?

The obvious question for anyone who's read a Dirty PageTable writeup: can you free the slab page back to buddy and reclaim it as a PTE page?

I spent a lot of time on this. The short answer is that I couldn't make it work, and the reasons are instructive.

`kmalloc-256` uses order-1 slabs (8,192 bytes, 32 objects). ARM64 PTE pages are order-0 (4,096 bytes). These live on separate PCP freelists. An order-1 page freed from the slab cache sits in the order-1 PCP list. A PTE allocation asks for order-0. The two don't mix unless PCP overflows into the buddy allocator, which then splits the order-1 page. That only happens under specific heap pressure that's hard to arrange during the race window.

I verified independently that the individual pieces of cross-cache work fine. With heap shaping (pad on a different CPU, flush `cpu_partial`), I could reliably drive 244 out of 250 slab pages to buddy. With 16 children forking and touching 8 GB each, all available UNMOVABLE order-1 pages get split for PTE allocations. The slab-to-buddy path works. The buddy-to-PTE path works.

But combining them with the race is a timing problem. The walker traverses 8,000 entries in about 2 ms. The cross-cache pipeline -- SLUB discard, buddy insertion, PTE allocation, `__GFP_ZERO` -- takes on the order of 100 ms. The gen write needs to land on a physical page that has already completed the entire transition from slab object to live PTE page. Those timelines don't overlap, and I couldn't find a way to make them overlap without privileged scheduler tricks that defeat the point of an unprivileged exploit.

Same-cache reclaim, staying inside `kmalloc-256`, sidesteps all of this. SLUB's per-CPU freelist is LIFO: last freed, first allocated. An immediate `kmalloc(256)` on the same CPU reclaims the exact slot. The challenge moves from "can we reclaim?" to "what useful `kmalloc-256` object has an exploitable layout at offsets 168 and 176?"

That's the real exploitation question, and I'm leaving it as an exercise.

---

## The Fix

[Commit 07712db80857](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=07712db80857d5d09ae08f3df85a708ecfc3b61f):

```diff
 static void ep_free(struct eventpoll *ep)
 {
     mutex_destroy(&ep->mtx);
     free_uid(ep->user);
     wakeup_source_unregister(ep->ws);
-    kfree(ep);
+    kfree_rcu(ep, rcu);
 }
```

A 16-byte `struct rcu_head` is added to `struct eventpoll`. `kfree_rcu()` defers the free until the current RCU grace period ends. Since the walker holds `rcu_read_lock()`, the grace period can't complete until it finishes. The `eventpoll` stays live for the entire traversal. Done.

---

## What I Keep Thinking About

The hardest part of finding this bug wasn't winning the race or understanding the slab allocator. It was building up enough of epoll's synchronization model in my head to know which paths are protected and which aren't. Wait queue locks serialize callbacks. File refcounts gate `ep_free`. `__fput` orders cleanup steps. `call_rcu` defers `epitem` frees. Each mechanism protects something specific. You have to hold all of them simultaneously before you can point at `epi->ep` and say: nothing covers this.

The 2023 optimization is a case study in a specific failure mode. The old `epmutex` was overbroad -- that's exactly why removing it gave 60%. But overbroad also means incidentally protective. The graph walkers were unintentional beneficiaries of the global lock. They weren't in anyone's audit of the change, because they don't touch the data the mutex was nominally guarding. They just follow pointers through structures that the mutex was keeping alive.

This generalizes. When you remove a lock that was held longer than it needed to be, you're not just removing contention. You're removing protection from every reader who was depending on that lock to keep something else alive, whether they knew it or not. Each of those readers needs to be re-audited. Even the ones that don't obviously touch the protected data. Especially those.

The fix is [upstream](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=07712db80857d5d09ae08f3df85a708ecfc3b61f). Update your kernels.
