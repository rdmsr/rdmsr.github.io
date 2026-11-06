+++
title = "FreeBSD GUS"
date = 2026-10-02
description = "RCU's cool cousin"

[extra]
type = "Post"
+++


## Introduction
FreeBSD has a pretty neat safe memory reclamation scheme (something akin to RCU), that's also been adopted by XNU that hasn't really been well-documented outside of code comments, and I thought it would've been interesting to write about it.

## What safe memory reclamation is all about
The problem of *safe memory reclamation* (SMR) shows up notably when working with lock-free data structures. Lock-free data structures are the bread and butter of modern scalability in operating systems, and they often allow much higher throughput than their finely grained locked counterparts, but they come with their own issues.

Lock-free data structure code in unmanaged languages needs to carefully manage the liveness of the objects it manages, i.e it must guarantee that memory is freed if and only if no one is able to access it. This arises because there is no synchronization around the access of the data structure, so an object that is removed can still be visible concurrently by another CPU (who might have made a local copy).

More broadly, SMR schemes can also be used for other things than memory reclamation, I like to think of them as providing barriers ensuring that all participants (read: CPUs or threads) have passed through a certain point.

## Prior art

SMR schemes broadly fall in two families: *Epoch-based reclamation* (EBR) and *Quiescent-state based reclamation* (QSBR). You might have also heard of *Hazard Pointers* (HP), which are useful on their own but are seldom seen in operating systems (at least, that I know of!).

QSBR is notable because you might've heard of it called a different name by the Linux project, *RCU* (Read-Copy-Update)! RCU's author, Paul McKenney, actually invented QSBR for the DYNIX operating system.

RCU has been described extensively, and I would recommend reading McKenney's excellent [LWN Articles](https://lwn.net/Articles/262464/) to learn more. The TL;DR is that there are three operations: `rcu_read_lock`, `rcu_read_unlock` and `rcu_synchronize`, the `read` operations are used to guard reads around potentially-freed memory (i.e to enter a *read section*) and the `synchronize` function waits until there is no CPU in a read section (this could be used to wait until memory is safe to be freed).

The gist of the algorithm is that memory is known to be safe to free when all CPUs have gone through at least one *quiescent point* ever since the object was removed from its data structure (a *grace period*), and read section operations block those quiescent points from ever happening. 

In practice, this can be implemented by having a quiescent point on context switches and inhibiting preemption during read operations. The core of the algorithm is actually pretty simple, and much of the complexity of the Linux code stems from making this scalable (Tree RCU) and handling other RCU flavors (e.g preemptible RCU).

FreeBSD's safe memory reclamation scheme is called *GUS* and is part of the EBR family.

## Epoch-based reclamation

Epoch-based reclamation was first described by Keir Fraser in his now-famous thesis titled *Practical Lock-freedom* ([link](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-579.pdf)). The way it works is that there is a global *epoch* which moves through 0, 1 and 2. When a thread enters a read section, it pins itself by reading the global epoch and copying it into its local slot. 

When a writer retires an object, it goes into a "limbo" bag tagged with the current epoch (really, the epoch is used as an index into an array of three linked lists). The global epoch is advanced by *polling* all other threads and checking whether or not they have acknowledged the current epoch by reading their local slot; if it so happens that they have, then the global epoch is advanced and memory that was retired two epochs ago can now be freed.

The main issues that arise:
1. Epochs are *bounded*, i.e they always cycle through 0, 1 and 2, this means that writers have to synchronize with epoch advancement before retiring more objects; and
2. Doing an O(n) scan touching remote memory is very expensive!

It is now time to reveal that GUS really means *Global Unbounded Sequences*, and it tackles on issue #1 by making epochs unbounded (hence the name) and issue #2 is somewhat mitigated by clever integration with the broader system.

## GUS

> I am using sequence and epoch interchangeably here. Confusing, I know!

The way GUS works is that there is a global, monotonically increasing write sequence, as well as a global read sequence, which represents the last minimum observed sequence across CPUs.

Each CPU also keeps track of its own local sequence when it is pinned, akin to how EBR does things.

The algorithm goes as follows, in pseudocode:

```
enter():
    store(cpu.cur_seq, load(global_write_seq, relaxed), relaxed)
    fence(seq_cst)

exit():
    store(cpu.cur_seq, SEQ_INVALID, release)

advance():
    fetch_add(global_write_seq, 1, release)

// Returns whether a sequence `goal` is reached
poll(goal):
    fence(seq_cst)
    
    if goal <= global_read_seq:
        // Goal already reached!
        return true

    min_seq = global_read_seq

    for each cpu:
        min_seq = min(min_seq, load(cpu.cur_seq, relaxed))

    if goal <= min_seq:
        // Goal reached across all CPUs
        fence(acquire)
        
        Exchange global_read_seq with the new one.
        return true
        
    return false
```

> I will not go *too* much into details about the atomic operations here, if you're curious, they have been extensively described by people smarter than me! The basic idea, however, is that one needs to remember that all operations after `enter` must not be reordered w.r.t the publication of the epoch (the store to `cur_seq`)

The basic idea is therefore that a sequence is essentially a timestamp[^1], and that the read sequence is always playing catch-up with the write sequence, but never actually reaching it,
(i.e there is always at least a separation of 1 between both). Obviously, in practice, there is a bit more to it especially w.r.t memory orderings, but the whole thing is still freakishingly simple 
and can be implemented in a few hundred lines, see the FreeBSD implementation [here](https://elixir.bootlin.com/freebsd/v15.1/source/sys/kern/subr_smr.c).

The unbounded epochs solved the issue #1 that was presented earlier, but what about issue #2: how is polling amortized?


Jeff Roberson, the author of GUS, also wrote the UMA (universal memory allocator) implementation, and both tie nicely together.

## Integration with the rest of the system
To me, the way GUS is integrated with the allocator is really its distinguishing feature and what makes the whole thing work well.

The UMA allocator is essentially a modernized slab allocator, which was first implemented in Solaris and described in Jeff Bonwick's influential papers[^2][^3] in the early 2000s.
To make things scale on SMP systems, it relies on per-CPU "buckets" (magazines in Bonwickspeak), which are essentially an array of *N* objects on which objects are popped from and pushed onto quickly.
A certain amount of pre-allocated buckets are stored in a per-zone (`kmem_cache` in Bonwickspeak) "depot" from which empty buckets and full buckets are pulled. *N*, i.e the size of buckets, is dynamically resized
according to measured contention on the depot lock; high contention needs to be reduced by increasing the bucket size, 
which provides amortization over the (potentially) slow and contended operation of going to the depot.

GUS and the allocator join together by allowing the creation of "SMR zones" which are zones that do careful handling around the freeing of objects.
Someone wanting to write a lock-free data structure could create a SMR zone for their object and do allocations and frees effectively transparently.

On SMR zones, when the allocator notices a bucket is completely free (i.e full of objects), it advances the write sequence and stamps the bucket with it, 
then puts it back at the end of the depot free list. 

When the allocator then needs to get a free bucket from the depot, it gets the bucket from the head of the free list, 
and **then** polls other CPUs for the sequence that is stamped on that bucket. If the goal has already been reached, then the bucket can be reused, otherwise, 
the allocator will fallback on allocating another bucket itself.

{{<figure src="/zone-depot.svg" alt="" caption="Diagram of the zone depot" width="100%"/>}}

What's cool about this is that buckets naturally provide amortization for those SMR operations;
polling is done on a batch of objects at once, and not on a per-object basis (which avoids starting one write sequence per object) and growing the buckets lengthens the time spent before touching the depot again, which gives the time for other buckets to age.

Remember that the GUS algorithm relies on finding the lowest sequence across all CPUs, which means that oftentimes, if buckets are aged as expected,
then a single poll operation could allow the reuse of many buckets at the same time, e.g if the depot free list contains buckets stamped with `100, 101, 102` and we notice via polling that the new minimum sequence is `103`, 
then all of those buckets can be reused immediately, which avoids having to do future polls for those buckets.

That is what makes everything click together and the polling not so bad anymore!


## XNU
Apple's XNU has also adopted a very similar scheme but with many goodies on top. For instance, they support preemptible read sections and a `call_rcu`-style API.
The way it integrates with their zone allocator, however, is quite interesting and its implementation differs enough that I deem it worthy to be described here as well.

XNU's zone allocator (`zalloc`) is, like UMA, also essentially a modernized slab allocator;
I will not try to explain it in depth here because it is quite sophisticated (over 10k lines!), and I am keeping this in reserve for another post, but I will go over the details of how their buckets are reused.

What's interesting is that zalloc does not rely on resizing buckets at all, instead, buckets are kept at a fixed size (something like 8 objects),
and a per-CPU depot is installed on top of the per-zone depot, which they call the "recirculation depot" (this should make sense later why such a name was used). 
This per-CPU depot is then resized with an algorithm similar to the one used to resize buckets, providing the same kind of amortization. 

When a bucket becomes full and ready to be recycled, it is stamped with a *deferred* advance of the write sequence and put onto the per-CPU depot.
When allocation is in need of a bucket, it polls the head of the per-CPU depot, and grabs it if it has expired. Otherwise, it does four things:

1. **Append all the buckets from the per-CPU depot to the zone depot**

Since the head of the per-CPU depot hasn't expired yet, this means that all buckets that sit in the per-CPU depot are not yet ready. 
All buckets are therefore put at the *end* of the zone depot, as to let them age behind the buckets that were already in depot.

2. **Commit the sequence of the last bucket in the zone depot**

The last bucket in the depot is the newest, its sequence is taken and the global write sequence is advanced to that one immediately,
this allows to do one write sequence write for many buckets.

3. **Pull back a bunch of buckets from the head of zone depot into the per-CPU depot**

A batch of buckets is then pulled from the head of the zone depot, these are the oldest buckets, so they are more likely to have expired.
The batching here avoids having to constantly thrash on the depot and avoids contention on it.

4. **Retry to allocate from per-CPU depot**

Finally, the oldest bucket we just freshly pulled from the zone depot is polled again, if it still hasn't expired, then
there is no choice but to fallback on allocating new memory.

What's all this jazz about deferring and committing? I like to think of a deferred advance as a "fake advance", it snapshots the current sequence but does not increment it,
which avoids a global read-modify-write (RMW) operation on the global write sequence. The key idea is that there is no need to advance the sequence, i.e commit it, until we actually need to, and
this also serves as an amortization feature because a sequence can be jumped to instantly, for instance, from 1 to 5 instead of individually advancing through 1, 2, 3, 4, and 5.

The recirculation depot is hence called as such because it allows a given bucket to *recirculate* and age before given the chance to be reused again.


{{<figure src="/recirc-depot.svg" alt="" caption="Diagram of the recirculation depot" width="100%"/>}}

## Pros & Cons
This SMR scheme, like anything else, has its own tradeoffs.
It is relevant to compare it with RCU because both are analogous, compared to RCU:

Pros:
- Only CPUs actively in a read-section need to participate during polling.
- Simple implementation
- Lower free-to-reuse latency[^4]

Cons:
  - Even though polling is amortized, it is still an expensive operation, and might even be a no-go on large NUMA systems.
  - Higher overhead on entering read sections (needs a fence)
  
  
## Resources
The best resource, as always, is reading the actual code, which is thankfully very well commented ([GUS link](https://elixir.bootlin.com/freebsd/v15.1/source/sys/kern/subr_smr.c), [UMA link](https://elixir.bootlin.com/freebsd/v15.1/source/sys/vm/uma_core.c)).
The XNU code is also heavily commented ([SMR link](https://github.com/apple-oss-distributions/xnu/blob/main/osfmk/kern/smr.c), [zalloc link](https://github.com/apple-oss-distributions/xnu/blob/main/osfmk/kern/zalloc.c)).

Additionally, these are useful resources:
- Jeff Roberson has made a short [talk](https://www.youtube.com/watch?v=ZXUIFj4nRjk) about GUS and UMA.
- Comparison with RCU by [Joel Fernandes](https://people.kernel.org/joelfernandes/gus-vs-rcu), a Linux developer.


I have also implemented GUS and an allocator inspired by XNU's and UMA in my own operating system, zag ([SMR link](https://github.com/rdmsr/zag/blob/master/src/sys/kern/smr.zig), [allocator link](https://github.com/rdmsr/zag/blob/master/src/mm/zone.zig))


## Footnotes

[^1]: In fact, one could use a timestamp counter such as the TSC for this, as seen in [*Parallel Sequences*](https://faculty.cs.gwu.edu/gparmer/publications/eurosys16ps.pdf).
[^2]: [*The Slab Allocator: An Object-Caching Kernel Memory Allocator*](https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf)
[^3]: [*Magazines and Vmem: Extending the Slab Allocator to Many CPUs and Arbitrary Resources*](http://www.parrot.org/sites/www.parrot.org/files/vmem.pdf)
[^4]: This was actually measured by the FreeBSD people, see the talk I mentioned above.
