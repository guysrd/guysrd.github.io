---
layout: default
---

# The Refactoring That Opened a Door

In March 2023, a performance optimization landed in the Linux kernel's
epoll subsystem. The global `epmutex`, a single mutex that serialized
all epoll topology operations, was replaced with per instance reference
counting and a `dying` flag. The motivation was sound: under HTTP
benchmark workloads, 58% of CPU time was spent in `osq_lock` contending
on that single mutex. The patch delivered a 60% throughput improvement.

But the old mutex had protected more than its authors realized.

## Epoll internals

`epoll` is Linux's scalable I/O event notification mechanism. A process
creates an epoll instance with `epoll_create`, adds file descriptors to
it with `epoll_ctl(EPOLL_CTL_ADD)`, and waits for events with
`epoll_wait`. Internally, the kernel maintains a `struct eventpoll` per
instance and a `struct epitem` per monitored fd. These are linked
through RB trees, hlists, and wait queues, forming a graph the kernel
must walk to detect cycles and limit nesting depth when epoll instances
monitor other epoll instances.

Epoll instances can watch other epoll instances. When you call
`epoll_ctl(EPOLL_CTL_ADD)` with a target that is itself an epoll fd,
the kernel enters a validation path to check for cycles and measure
nesting depth. Two functions do this walking:

- `reverse_path_check_proc` walks a monitored file's watcher list,
  counting wakeup paths. For each watcher, it reads `epi->ep` to
  reach that watcher's eventpoll. Recursive, up to `EP_MAX_NESTS` (4)
  levels.

- `ep_get_upwards_depth_proc` walks an eventpoll's `refs` hlist,
  measuring chain depth going upward. Also reads `epi->ep` at each
  step. This function both reads and writes: it deposits
  `loop_check_gen` (a global counter) and a depth result into the
  reached eventpoll.

Both walk linked lists under `rcu_read_lock`. Both dereference
`epi->ep`:

```c
/* fs/eventpoll.c, called from ep_loop_check() */
static int ep_get_upwards_depth_proc(struct eventpoll *ep, int depth)
{
    int result = 0;
    struct epitem *epi;

    if (ep->gen == loop_check_gen)
        return ep->loop_check_depth;
    hlist_for_each_entry_rcu(epi, &ep->refs, fllink)
        result = max(result, ep_get_upwards_depth_proc(epi->ep, depth + 1) + 1);
    ep->gen = loop_check_gen;
    ep->loop_check_depth = result;
    return result;
}
```

```c
static int reverse_path_check_proc(struct hlist_head *refs, int depth)
{
    int error = 0;
    struct epitem *epi;

    if (depth > EP_MAX_NESTS)
        return -1;

    hlist_for_each_entry_rcu(epi, refs, fllink) {
        struct hlist_head *refs = &epi->ep->refs;
        if (hlist_empty(refs))
            error = path_count_inc(depth);
        else
            error = reverse_path_check_proc(refs, depth + 1);
        if (error != 0)
            break;
    }
    return error;
}
```

This will matter.

## The change

The refactoring replaced `epmutex` with a per ep `refcount_t` on
`struct eventpoll` and a `dying` flag on each `struct epitem`. The new
scheme correctly handles the race between `ep_clear_and_put` (closing
an epoll fd) and `eventpoll_release_file` (closing a file that epoll
monitors). That interaction was carefully audited. But the graph
walking functions above were not. They hold `epnested_mutex` (the
renamed, narrower successor to `epmutex`). The new teardown path,
`ep_clear_and_put`, does not. They share no lock.

Here's the new teardown. It holds only the per instance `ep->mtx`:

```c
static void ep_clear_and_put(struct eventpoll *ep)
{
    mutex_lock(&ep->mtx);

    for (rbp = rb_first_cached(&ep->rbr); rbp; rbp = next) {
        next = rb_next(rbp);
        epi = rb_entry(rbp, struct epitem, rbn);
        ep_remove_safe(ep, epi);
        cond_resched();
    }

    mutex_unlock(&ep->mtx);
    if (ep_refcount_dec_and_test(ep))
        ep_free(ep);
}
```

And `ep_free` just frees the struct directly:

```c
static void ep_free(struct eventpoll *ep)
{
    mutex_destroy(&ep->mtx);
    free_uid(ep->user);
    wakeup_source_unregister(ep->ws);
    kfree(ep);
}
```

The `dying` flag and refcount correctly handle the race between
`ep_clear_and_put` and `eventpoll_release_file`. That race was audited.
But the graph walking functions were not. They hold `epnested_mutex`.
The teardown path does not. They share no lock.

## The bug

It iterates the `refs` hlist under `rcu_read_lock`. For each epitem,
it reads `epi->ep` and recurses.

The epitem itself is safe, it's freed via `call_rcu`:

```c
/*
 * At this point it is safe to free the eventpoll item. Use the union
 * field epi->rcu, since we are trying to minimize the size of
 * 'struct epitem'. The rcu read side, reverse_path_check_proc(),
 * does not make use of the rbn field.
 */
call_rcu(&epi->rcu, epi_rcu_free);
```

The comment even acknowledges the RCU reader. But it only addresses
whether the epitem's memory is valid. It says nothing about the
eventpoll the epitem points to.

And `ep_free` uses `kfree(ep)`, not `call_rcu`, not `kfree_rcu`.
The eventpoll is freed immediately, without waiting for the RCU grace
period. If anyone is reading `epi->ep` under `rcu_read_lock` at that
moment, they're reading freed memory.

This is the rule `call_rcu` follows: it protects the object you call
it on. It does not protect objects reachable through pointers in that
object. `call_rcu(epi)` keeps `epi` alive. It says nothing about
`epi->ep`.

## Winning the race

The window between loading `epi` from the hlist and dereferencing
`epi->ep` is narrow. From the disassembly of `ep_get_upwards_depth_proc`
on an arm64 GKI kernel:

```
ffffffc08042fa64:  ldr x9, [x19, #176]    ; x9 = ep->refs.first
ffffffc08042fa70:  sub x23, x9, #0x50     ; x23 = container_of(x9, epitem, fllink)
    ...
ffffffc08042fa80:  ldr x0, [x23, #72]     ; x0 = epi->ep  (the stale pointer)
ffffffc08042fa88:  bl  ep_get_upwards_depth_proc  ; recurse into freed memory
```

Inside the recursive call, the first dereference of the freed pointer:

```
ffffffc08042fa48:  ldr x9, [x0, #168]     ; read ep->gen from freed slot
```

This is the crash site. From pstore after a successful trigger:
`pc : ep_get_upwards_depth_proc+0x1c/0xa4`.

On a different CPU, hitting the window between `ldr x0, [x23, #72]`
and the `bl` is nearly impossible. Cache coherence latency alone
exceeds the gap.

The technique that works is same CPU preemption. Pin both threads to
the same core. The main thread enters the kernel and starts walking the
hlist. A timer tick fires during the traversal. On a `PREEMPT_RCU`
kernel, `rcu_read_lock` does not disable preemption. It just
increments a counter. The scheduler preempts the main thread. The
closer thread runs, frees the eventpoll, reclaims the slot, and
yields. The main thread resumes and dereferences the stale pointer.

For this to work, the traversal has to last long enough for a timer
tick to happen. At `CONFIG_HZ=250`, a tick fires every 4 milliseconds.
With 4,096 epoll parents, the traversal takes about 400 microseconds,
too short. At 8,000 parents, it's around 2 milliseconds, which
reliably overlaps a tick boundary. Hit rate: about 4% per iteration.

There's one more subtlety. CFS (the Completely Fair Scheduler) tracks
virtual runtime. A thread that busy waits while waiting for the signal
accumulates high vruntime. When the tick fires, CFS might prefer the
main thread (lower vruntime) over the closer, exactly the wrong
priority. The counterintuitive fix: the closer *sleeps*.
`usleep(1000)` freezes its vruntime. When it wakes, CFS sees a thread
with almost no accumulated runtime and strongly prefers it over the
main thread.

Here's what the race looks like, with both threads on CPU 0:

```
Thread A (main, lower priority)         Thread B (closer, wakes with low vruntime)

epoll_ctl(ep_a, ADD, ep_t)
  ep_loop_check()
    rcu_read_lock()
    ep_get_upwards_depth_proc(ep_a, 0)
      hlist iteration: load epi
      epi->ep points to ep_b
      |
      |  <-- timer tick fires here
      |  <-- scheduler preempts thread A
      |
      |                                 wakes from usleep (low vruntime, CFS prefers us)
      |                                 close(ep_b_fd)
      |                                   ep_clear_and_put(ep_b)
      |                                     __ep_remove: hlist_del_rcu(epi)
      |                                     call_rcu(epi)        -- epi deferred, safe
      |                                   ep_free(ep_b)
      |                                     kfree(ep_b)          -- ep freed NOW
      |                                 kmalloc(N)               -- reclaims the slot
      |                                 done, yields
      |
      resumes
      ep_get_upwards_depth_proc(epi->ep)
        epi->ep is the old address of ep_b
        slot now contains the new object's data
        reads ep->gen at offset 168
        reads ep->refs.first at offset 176
        if refs.first is NULL: writes loop_check_gen at offset 168
                               writes 0 at offset 184
        if refs.first is nonzero: follows it as a pointer...
```

The critical moment is the reclaim. The closer frees the eventpoll,
then immediately allocates a new object in the same kmalloc bucket.
SLUB's per CPU freelist is LIFO: last freed, first allocated. The new
object lands in the exact slot the eventpoll just vacated. When the
main thread resumes, it reads the new object's data where it expects
eventpoll fields.

## The refs.first problem

Whether this crashes or not depends entirely on one field:
`eventpoll.refs`, the hlist head at offset 176 of the struct.

From `pahole`:

```
struct eventpoll {
    ...
    struct file *              file;                 /*   160     8 */
    u64                        gen;                  /*   168     8 */
    struct hlist_head          refs;                 /*   176     8 */
    u8                         loop_check_depth;     /*   184     1 */
    ...

    /* size: 200, cachelines: 4 */
};
```

Look at the function again:

```c
hlist_for_each_entry_rcu(epi, &ep->refs, fllink)
    result = max(result, ep_get_upwards_depth_proc(epi->ep, depth + 1) + 1);
ep->gen = loop_check_gen;
ep->loop_check_depth = result;
```

If `refs.first` is NULL, the hlist is empty. The loop body never
executes. The function writes `loop_check_gen` at offset 168 and
the depth byte at offset 184, then returns. No pointer is followed.
No crash. A controlled, silent corruption.

If `refs.first` is nonzero, the kernel treats it as a pointer to a
`struct epitem` and follows a chain of dereferences through kernel
memory. On a freed and reallocated slot, that value is whatever the
new object has at offset 176. If it looks like a valid kernel address,
the function recurses into it. If it doesn't, the kernel faults
and panics.

This is the central challenge for exploitation. You need the reclaim
object to have zero at offset 176. Not all objects do. On a kernel
with `init_on_free=1`, a freshly freed slot is zeroed before reuse, so
the window right after kfree has all zeros including offset 176. But if
the new object writes nonzero data at that offset during its
initialization, the zero is gone by the time the main thread reads it.

Finding an object that lets an attacker choose what will be at offset
176 is not trivial. `kmalloc-256` does not have many general purpose
objects with attacker controlled content at arbitrary offsets. Most
kernel structs that land in this bucket have fixed layouts where offset
176 corresponds to a pointer field or a lock, initialized to nonzero
values by the kernel. Yet it is doable. The right reclaim object exists,
and once found, the function becomes a multilevel read/write gadget
through kernel memory: it follows `refs.first` as a pointer, reads and
writes at fixed offsets, and recurses. Each level deposits
`loop_check_gen` (a partially controllable counter) and a depth byte
(zero). With an infoleak, this turns into an arbitrary read/write
interface. Without one, you need to think about how to turn this bug
into a read primitive too, or find an independent infoleak elsewhere.

## The cross cache wall

A natural idea is to free the eventpoll's entire slab page back to the
page allocator and reclaim it as a different slab cache, one where offset
176 is guaranteed zero and offset 168 overlaps a critical field.
Pipe buffer rings (`kmalloc-192`) or page table entries are classic
cross cache targets.

This is not trivial on modern kernels, for several reasons.

First, `struct eventpoll` lives in `kmalloc-256`, which uses order 1
slabs (8 KB, two contiguous pages). Pipe buffer rings live in
`kmalloc-192`, which uses order 0 slabs (4 KB, one page). The page
allocator's per CPU page cache (PCP) maintains separate freelists per
page order. An order 1 page freed by kmalloc-256 cannot serve an
order 0 allocation from kmalloc-192. The page sits on the PCP's order 1
list until something else requests an order 1 page. On the devices
tested, the PCP high watermark was 845 pages, far above what a single
exploit attempt could fill. The freed pages never reached the buddy
allocator where they could be split.

Second, `init_on_free=1` zeros the slab slot immediately on `kfree`,
before the object reaches the freelist. But the SLUB freelist pointer
is written into the slot *after* the zeroing (at `s->offset`, which is
offset 0 for `kmalloc-256` on tested kernels). Any cross cache reclaim
object that gets allocated on the page would then be initialized by its
own constructor, overwriting the zeroed bytes. The narrow window where
offset 176 is still zero from `init_on_free` closes before the main
thread can read it, because the cross cache path goes through PCP and
buddy, which is orders of magnitude slower than same cache SLUB LIFO.

Third, page table entries (the "dirty pagetable" technique) have the
same order mismatch. PTE pages are order 0 on arm64. The freed
order 1 kmalloc-256 slab cannot become a PTE page without a buddy
split, which requires PCP overflow that does not happen in practice.

Same cache reclaim (staying within `kmalloc-256`) avoids all of these
problems. SLUB LIFO on the same CPU gives immediate, reliable slot
reuse. The challenge moves from "can we reclaim the slot" to "can we
find a useful object in the same bucket."

## The fix

```diff
 static void ep_free(struct eventpoll *ep)
 {
     mutex_destroy(&ep->mtx);
     free_uid(ep->user);
     wakeup_source_unregister(ep->ws);
-    kfree(ep);
+    /* ep_get_upwards_depth_proc() may still hold epi->ep under RCU */
+    kfree_rcu(ep, rcu);
 }
```

`kfree_rcu` defers the free until after the current RCU grace period
completes. The graph walking functions run under `rcu_read_lock`, which
prevents the grace period from completing while they hold references.
The eventpoll stays valid for the entire read side critical section.
The `struct rcu_head rcu` field (16 bytes) is added to `struct
eventpoll` for the RCU callback infrastructure.

I built both versions and tested them in QEMU. The kernel with
`kfree_rcu`: 500 iterations of the trigger, zero crashes. The same
kernel with `kfree` instead: crashes reliably within 200 iterations.

## Final Words

The hardest part wasn't finding the bug. It was understanding epoll's
synchronization model well enough to know which paths are actually
unprotected. Wait queue locks serialize callbacks. File refcounts gate
`ep_free`. `__fput` orders cleanup steps in a specific sequence. You
have to internalize all of that before you can look at the RCU read
side traversal of `epi->ep` and say with confidence: nothing protects
this.

If you're up for a challenge, try exploiting this on a modern Android
system from an untrusted app context. This bug gives you a write primitive. 
Turning it into a reliable PE is a different problem.
