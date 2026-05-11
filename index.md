---
layout: default
---

# When an Optimization Opens a Door

## TL;DR

In March 2023, a Linux kernel patch optimized the `epoll` subsystem by replacing a global mutex with per-instance reference counting. The patch delivered a 60% throughput improvement on HTTP benchmarks. But the old mutex had been silently protecting code paths the authors didn't consider. The result: a use-after-free on a kernel pointer, reachable from any unprivileged process that can call `epoll_ctl`.

This post walks through how the bug came to exist, why it's a UAF (and not a different bug class), why exploiting it on a hardened kernel is much harder than it sounds, and the one-line fix that closed it.

*This post assumes familiarity with the Linux kernel's memory allocator and RCU.*

---

## A 60-Second Primer on epoll

If you've used Linux servers, you've benefited from `epoll` without seeing it. It's the kernel's scalable I/O event notification system — the thing that lets nginx, redis, and node.js watch tens of thousands of sockets without blocking.

Three syscalls do all the work:

- `epoll_create()` — make a new epoll instance, get back a file descriptor.
- `epoll_ctl(ADD / MOD / DEL)` — start, modify, or stop watching some other file descriptor.
- `epoll_wait()` — block until one of the watched fds has activity.

The interesting twist: an epoll fd is itself a file descriptor, so you can add an epoll fd to *another* epoll instance. This creates a directed graph of epoll instances watching epoll instances. To prevent the kernel from looping forever — or from blowing the stack on a deeply nested chain — there is graph-walking validation code that runs inside `epoll_ctl(ADD)` to check for cycles and enforce a nesting depth limit.

**That validation code is where the bug lives.**

---

## The Two Structures You Need to Know

There are two kernel objects in play. A diagram first, then the words.

![epoll data structures and the UAF](1.svg)

**`struct eventpoll`** — one per epoll instance (one per `epoll_create()` call). Holds the wait queue, the RB tree of items it's currently watching, and — critically for us — `refs`, an hlist head linking every `epitem` that points *back at this instance from somewhere else*.

**`struct epitem`** — one per (epoll instance, watched fd) pair. Holds `epi->ep`, a pointer back to its owning `eventpoll`. If the watched fd is itself an epoll, this `epitem` is also linked into *that* epoll's `refs` hlist via `fllink`.

So when the kernel needs to walk the graph "upward" — from a child instance to all the parents watching it — it iterates the `refs` hlist, and for each `epitem` it finds, it follows `epi->ep` to reach a parent `eventpoll`. Then it recurses.

That recursion — that *exact* dereference, `epi->ep`, inside that loop — is the UAF.

---

## The 2023 Refactoring

Before March 2023, every `epoll_ctl(ADD)` involving the nested case acquired a single global mutex called `epmutex`. Under HTTP benchmark workloads, 58% of CPU time was spent in `osq_lock` contending on it.

The patch did three things:

1. Added a per-instance `refcount_t` to `struct eventpoll`.
2. Added a `dying` flag to `struct epitem`.
3. Renamed the now-narrow lock to `epnested_mutex`, held only when the graph actually needs walking.

Result: **60% throughput improvement.** A genuine win.

The patch authors carefully audited the race between two specific paths:

- `ep_clear_and_put()` — runs when an epoll fd is closed.
- `eventpoll_release_file()` — runs when a watched file is closed.

That race is correctly handled by the new refcount + dying flag. **But there was a third path that used to be protected by the old global mutex, and now wasn't.** Nobody noticed for over a year.

---

## The Bug: a UAF on `epi->ep`

Here's the vulnerable function. It's the "upward" graph walker, called during `epoll_ctl(ADD)` to compute nesting depth:

```c
static int ep_get_upwards_depth_proc(struct eventpoll *ep, int depth)
{
    int result = 0;
    struct epitem *epi;

    if (ep->gen == loop_check_gen)
        return ep->loop_check_depth;

    hlist_for_each_entry_rcu(epi, &ep->refs, fllink)
        result = max(result, ep_get_upwards_depth_proc(epi->ep, depth + 1) + 1);
                                                       /* ^^^^^^^ UAF here */
    ep->gen = loop_check_gen;
    ep->loop_check_depth = result;
    return result;
}
```

This runs under `rcu_read_lock()`. The `epitem` itself is safe to read — when an `epitem` is unlinked, it's freed via `call_rcu()`, so RCU keeps it alive for the duration of any read-side critical section:

```c
/* The rcu read side, reverse_path_check_proc(), does not make use of the rbn field. */
call_rcu(&epi->rcu, epi_rcu_free);
```

The comment even acknowledges the RCU reader. But it only proves the `epitem`'s memory is alive. It says nothing about the `eventpoll` the `epitem` points to.

Now look at the teardown for a closed epoll instance:

```c
static void ep_free(struct eventpoll *ep)
{
    mutex_destroy(&ep->mtx);
    free_uid(ep->user);
    wakeup_source_unregister(ep->ws);
    kfree(ep);   /* immediate free — no RCU grace period */
}
```

`kfree()`. Not `kfree_rcu()`. **The `eventpoll` is freed immediately.**

### The rule that was missed

`call_rcu(epi)` keeps `epi` alive. It says nothing about `epi->ep`.

A pointer being safe to *read* is not the same as what it points to being safe to *use*. The reader is holding `epi` under RCU (good), reads `epi->ep` (still good — it's just a pointer load), then dereferences it (*not good*, because nothing keeps that target alive).

---

## The Race, Visualized

Two threads, both pinned to CPU 0. Thread A is inside `epoll_ctl(ADD)`, mid-walk through the graph. Thread B closes a different epoll instance at the wrong moment.

![The race timeline](2.svg)

`epi` is fine — RCU protects it. `epi->ep` points to a now-freed `eventpoll` slot, which has been instantly reused by a different allocation in the same slab cache. The function reads it as if it were still an `eventpoll`, follows pointers inside it, and writes back to it. **That's the use-after-free.**

---

## Why This Race Is Hard to Trigger

The window between loading `epi` from the hlist and dereferencing `epi->ep` is a handful of instructions. Across two physical CPUs, cache coherence latency alone is longer than that gap — the cross-CPU race is essentially unwinnable.

The trick is **same-CPU preemption.**

Under `PREEMPT_RCU` (default on Android/GKI kernels), `rcu_read_lock()` does *not* disable preemption. It just bumps a counter. So a timer tick during the walk can yield the CPU to the closer thread, even while the walker is mid-RCU.

The numbers, on the tested target:

- `CONFIG_HZ = 250` → a tick every 4 ms.
- 4,096 epoll parents → walk takes ~400 µs → too short, rarely hits a tick boundary.
- **8,000 epoll parents → walk takes ~2 ms → reliably overlaps a tick. Hit rate ~4% per iteration.**

There's one more subtle scheduler trick. CFS prefers threads with low virtual runtime. If the closer thread busy-waits for the signal, it accumulates high vruntime and CFS *de*prioritizes it — the opposite of what we want. The counterintuitive fix: `usleep(1000)`. When the closer wakes, its vruntime is near zero, and CFS strongly prefers it over the walker. Exactly the priority order needed.


---

## Same-Cache vs. Cross-Cache

Before going further into what to do with the freed slot, an obvious question every kernel exploit dev asks: can we cross-cache this? Free the slab page, reclaim it as something completely different — a pipe buffer ring, a page table page — where the layout is known and friendlier?

**No, not easily, for three reasons:**

1. **Order mismatch.** `struct eventpoll` is 200 bytes and lives in `kmalloc-256`, which uses **order-1 slabs (8 KB)**. Pipe buffer rings (`kmalloc-192`) and arm64 PTE pages are **order-0 (4 KB)**. The per-CPU page cache keeps separate freelists per order. An order-1 page won't satisfy an order-0 request unless it goes through the buddy allocator and gets split — and that only happens if the PCP overflows, which it doesn't in any realistic single-shot exploit (PCP high watermark was ~845 pages on tested devices).

2. **`init_on_free` zeroing window is too narrow.** On hardened kernels, `init_on_free=1` zeros a slot the moment it's freed. There *is* a window where `eventpoll`'s offset 176 is zero. But the cross-cache path (PCP → buddy → new cache → constructor) is orders of magnitude slower than same-cache SLUB LIFO, and any reclaim object will re-initialize the bytes at offset 176 long before the walker resumes.

3. **PTE cross-cache** ("dirty pagetable") fails for the same order-mismatch reason on arm64.

**Same-cache reclaim — staying in `kmalloc-256` — sidesteps all of this.** SLUB's per-CPU freelist is LIFO: last freed, first allocated. An immediate `kmalloc(N)` on the same CPU reclaims the *exact slot* that just held the `eventpoll`. The hard part shifts from "can we reclaim the slot?" to "is there a useful `kmalloc-256` object whose layout, at the offsets the walker reads, is exploitable?"

That's the real exploitation question.

---

## The `refs.first` Coin Flip

The walker is going to touch these fields of `struct eventpoll`:

| offset | field              | walker action     |
|--------|--------------------|-------------------|
| 168    | `gen`              | read, then write  |
| 176    | `refs.first`       | read as pointer   |
| 184    | `loop_check_depth` | write             |

The bytes at **offset 176** in the reclaim object decide everything:

- **If those bytes are zero** → hlist looks empty → walker skips the loop, writes `gen` and `depth`, returns. No crash. Silent corruption of 9 bytes (8 + 1) at offsets 168 and 184 of whatever object now lives there.
- **If those bytes are nonzero** → walker dereferences them as a pointer to an `epitem`, then dereferences *that* epitem's `ep`, and recurses. On garbage, the kernel panics.

Finding a `kmalloc-256` object where an attacker can control offset 176 (to a chosen pointer) *and* offset 168 (to a chosen value) turns this UAF into a recursive read/write gadget through kernel memory. Each recursion level deposits `loop_check_gen` (a partially attacker-influenced counter) and a zero byte at chosen offsets. Compose that with an info leak and you have arbitrary read/write. Compose it without one — that's the homework problem.

---

## The Fix

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

A 16-byte `struct rcu_head rcu` field is added to `struct eventpoll`. `kfree_rcu()` defers the free until the current RCU grace period ends. Since the walker is inside `rcu_read_lock()`, the grace period can't complete until it's done. The `eventpoll` stays valid for the entire traversal. Bug closed.

---

## What This Story Is Really About

The hardest part of finding this bug wasn't reading assembly or winning the race. It was internalizing epoll's synchronization model deeply enough to *know which paths aren't protected.* Wait queue locks serialize callbacks. File refcounts gate `ep_free`. `__fput` orders cleanup steps. Each of these protects something. You have to know all of them before you can confidently say: *nothing covers `epi->ep` here.*

The 2023 refactoring is a textbook case of an optimization that audits the obvious races and misses the non-obvious one. The old `epmutex` was overbroad — that's exactly why removing it gave 60% — but "overbroad" also means *incidentally protective*. The graph walkers were unintentional beneficiaries of the global lock. When the lock narrowed, the protection vanished, and the path didn't even register as "now unsynchronized" in anybody's review.

This is the lesson worth taking home: when you remove a lock that was held longer than it strictly needed to be, you are not just removing contention. You are removing protection from every reader who was, knowingly or not, depending on that lock to keep something else alive. Every one of them needs to be re-audited. Even the ones that don't touch the data the lock was nominally protecting.

If you're up for it: this bug gives you a write primitive. Turning it into a reliable privilege escalation on a modern Android system from an untrusted app context is a different problem entirely. Have fun.
