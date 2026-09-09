+++
title = "Solaris Turnstiles"
date = 2026-09-08

[extra]
type = "Post"
toc = true
+++


## Introduction
Sun microsystems' Solaris was once widely regarded as having some of the best symmetric multiprocessing (SMP) support among the operating systems of its time.
Much of this technical strength came from innovations developed within the project.


Although Solaris is now mostly defunct [^1], its influence remains substantial; technologies pioneered by Solaris can still be found across a [wide](https://en.wikipedia.org/wiki/ZFS) [range](https://en.wikipedia.org/wiki/Slab_allocation) of [software](https://en.wikipedia.org/wiki/DTrace)

A lot of Solaris' inventions have been described and talked about *ad nauseam* (such as the Slab Allocator), but one I rarely see discussed is its use of **turnstiles**.

Despite being relatively obscure, the idea has quietly spread far beyond Solaris. Variations of it can now be found in major operating systems, web browsers, and language runtimes. In fact, you’re probably using several implementations of the same basic concept right now!


## Why turnstiles exist
Before going in depth on turnstiles, let's take a step back and figure out what problems Sun engineers were trying to solve.


One of the key design hallmarks of Solaris was that it made heavy use of blocking mutexes (or locks), in part because they could provide much better latency for high-priority tasks, which is important if you wanted something approaching soft real-time behavior.


The catch was that blocking mutexes came with problems of their own, two of which turnstiles were designed to address.

### Mutexes are big
A blocking mutex needs more than just a single bit saying whether it’s locked. Somewhere, the kernel also needs to keep track of things like:

- Who currently owns the lock;
- which threads are waiting for it; and
- The state needed to coordinate blocking and waking those threads.

You could store all of this directly in every mutex, but that gets expensive pretty quickly.

Highly scalable software tends to rely on fine-grained locking, i.e having lots of locks protecting relatively small pieces of state. The problem is that most of those locks will be uncontended most of the time, so dedicating a large amount of bookkeeping to every single one is mostly wasted space.

Keeping mutexes small makes fine-grained locking much cheaper. If adding another lock only costs a handful of bytes, developers can afford to use more of them instead of combining unrelated state behind a smaller number of coarse-grained locks.


### Priority Inversion
Another significant problem that arises with the heavy use of blocking locks is *priority inversion*.


Typically, when a thread acquires a spinlock, it disables preemption (via mechanisms like `spl`, `Irql` or `preempt_disable`).
Disabling preemption effectively puts the thread at the highest priority on the system, blocking any other task that might want to interrupt its work from doing so until the spinlock is released.

This is great for throughput, but latency can suffer if preemption stays disabled for too long.

Blocking locks, on the other hand, do not disable preemption, which allows for better latency behavior; high priority tasks can interrupt other lower priority tasks. This can also reduce throughput however, so there is no single best solution (though this can in part be worked around through the use of *adaptive spinning*)


Since threads keep their priority when holding a lock, an especially ugly situation can occur: What if high priority task A tries to acquire a lock currently held by low priority task B?

A blocks, waiting for B to release the lock. So far, so good. The problem is that B is still a low-priority task. Any medium-priority task that becomes runnable can preempt B, preventing it from making progress and releasing the lock.

In effect, our high-priority task is now stuck waiting behind work that should never have been able to delay it in the first place. This is called *priority inversion*, and in the worst case it can delay A for an unbounded amount of time.

So much for those latency guarantees!

### Priority inheritance

Luckily, a bunch of smart people figured out a neat solution to this: **priority inheritance**.

In the same situation, B would temporarily *inherit* A's priority until it finishes its critical section, allowing it to run ahead of any medium-priority task that might otherwise get into its way.

At first glance, this seems pretty simple: when a high-priority thread blocks on a lock, boost the priority of whoever owns it. Things get more interesting, though, once locks start depending on other locks.

Consider the following scenario: A waits on B, but B itself is waiting on C. A could propagate its priority to B, but it would still have to wait for a potentially lower-priority C to release the lock. To correct this, A's priority needs to be propagated through the *owner chain* until it reaches C. This is called *multi-hop* priority inheritance.

What gets tricky is keeping track of these chains efficiently in the kernel.

Different operating systems have come up with different machinery for keeping track of these dependency chains.
Turnstiles are the mechanism Solaris and a bunch of UNIX-derived operating systems use to do exactly that.

A few other notable approaches are:

- AutoBoost on Windows; and
- Linux’s rt-mutex priority-inheritance machinery.

## Turnstiles


```
⠀⠀⠀⠀⠀⢀⣀⣀⣀⣀⣀⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⢀⣴⣿⣿⣿⣿⣿⣿⣿⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⢀⣴⣿⣧⣤⣤⣤⣤⣼⣿⠟⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⣿⡿⢋⣴⣿⠀⣶⣶⣶⣶⣶⣶⣶⣶⣶⣶⣶⣶⣶⣶⡄⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠋⠰⠿⠟⠃⣀⣉⡉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠀⠀⠰⣦⡄⠉⠻⢿⣷⣦⣀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠀⠀⠀⢻⣿⡄⠀⠀⠈⠙⠻⣿⣶⣤⣀⠀⠀⠀⠀⠀⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠀⠀⠀⠀⢿⣷⡀⠀⠀⠀⠀⠀⠙⠻⢿⣷⣦⣄⠀⠀⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠀⠀⠀⠀⠈⣿⣷⠀⠀⠀⠀⠀⠀⠀⠀⠈⠙⠛⠀⠀⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠀⠀⠀⠀⠀⠘⣿⣧⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠀⠀⠀⠀⠀⠀⠸⣿⣇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠀⠀⠀⠀⠀⠀⠀⠹⡿⠂⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⢸⣿⣿⣿⣿⣿⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠈⠉⠉⠉⠉⠉⠉⠀⠀
```


At a high level, a turnstile is a data structure associated with a contended lock. It keeps track of the threads waiting on that lock, along with the information needed to propagate priority through the lock’s owner chain.

The clever part is that this state doesn’t have to live inside the lock itself.

Instead, each thread gets its own turnstile when it is created, in case it might contend on a blocking lock. This hinges on the fact that a thread can only be blocked on one lock at a time. 

When a thread does block, it effectively donates its turnstile to the lock it is waiting on. That turnstile then becomes the place where the kernel keeps the lock’s waiters and priority-inheritance state.

This raises an obvious question: if the turnstile doesn’t live inside the lock, how does the kernel find it?

The answer is a hash table. The lock's address is hashed and is used to locate the turnstile currently associated with that lock. Keep in mind that the diagram below logically shows the hash table as having one turnstile per hash bucket, but in practice each bucket contains a chain of them: finding the appropriate turnstile then requires iterating over that bucket's chain to find the turnstile that points to the desired lock.

When a thread blocks on an already contended lock, it donates its turnstile onto a per-turnstile free list.
When the lock is finally released, each thread simply grabs a turnstile from the freelist, it does not matter which; a turnstile is not bound to a specific thread.

Each thread keeps track of the turnstile it is currently blocked on, and each turnstile knows the current owner of the lock.


{{<figure src="/turnstile.svg" alt="" caption="Turnstile diagram" width="100%"/>}}

### Owner chains

To do *multi-hop* priority inheritance, the priority boosting code needs to follow each successive lock dependency. Conceptually, this amounts to walking a chain that looks like:
`turnstile->owner->turnstile->owner...` until it reaches an owner that is not blocked on a synchronization object (this could be kept track by a `waiting_on` field in the thread structure).


The locking required to do this actually gets quite nasty, so I advise you read the comments in the [Illumos implementation](https://github.com/illumos/illumos-gate/blob/master/usr/src/uts/common/os/turnstile.c)[^2] if you're hungry for more details. The gist of it is that there is a `bucket lock -> thread lock` locking hierarchy, and the successive bucket locks are trylocked. If the acquisition fails then everything is dropped and tried again as to avoid a livelock between two CPUs currently doing a PI walk.


### Limitations

Keen observers may have noticed a glaring limitation with this scheme: a turnstile has a single inheritor. So what about locks with multiple owners, such as reader-writer locks?

This is where things get awkward. If a writer is blocked behind several readers, there is no single owner for it to donate its priority to.

Instead, kernels that use turnstiles rely on heuristics or "good enough" solutions, such as picking the first thread that acquired the lock as the inheritor, or straight up giving up on priority inheritance for multi-owner locks altogether. 
Another possible solution is simply boosting a thread's priority to some given ceiling before taking a shared lock.


I believe that the only priority inheritance scheme that *does* support multiple inheritors is NT's AutoBoost, but its internals are not widely documented.


## Influence
Many UNIX-derived operating systems started taking (copying) ideas from Solaris once they moved towards SMP support. As such, turnstiles are now found in many kernels of the same family.

Turnstiles are currently used in:
- Solaris, where they originated;
- [Illumos](https://github.com/illumos/illumos-gate/blob/master/usr/src/uts/common/os/turnstile.c);
- macOS's kernel, [XNU](https://github.com/apple-oss-distributions/xnu/blob/main/osfmk/kern/turnstile.c);
- [FreeBSD](https://github.com/freebsd/freebsd-src/blob/main/sys/kern/subr_turnstile.c); and
- [NetBSD](https://github.com/NetBSD/src/blob/trunk/sys/kern/kern_turnstile.c).

Additionally, some hobby OSes have adopted them:
- My own, [zag](https://github.com/rdmsr/zag/blob/master/src/kern/turnstile.zig)[^3]
- [MINTIA](https://github.com/xrarch/mintia2/blob/main/OS/Executive/Ke/KeTurnstile.jkl)
- [Keyronex](https://github.com/Keyronex/Keyronex/blob/master/kernel/common/kern/turnstil.c)


### Outside of operating systems
Interestingly, the same basic idea shows up outside of operating-system kernels as well. A few notable examples are:
- [Go's runtime semaphores](https://github.com/golang/go/blob/master/src/runtime/sema.go)
- [WTF::ParkingLot](https://webkit.org/blog/6161/locking-in-webkit/)

`WTF::ParkingLot` is particularly interesting because it is widely used and looks remarkably similar to the space-saving half of turnstiles.

Just like a turnstile, a parking lot keeps the expensive waiter machinery outside of the lock itself. When a thread needs to block, the address of the lock is used to find the corresponding wait queue in a global table.

The important difference is that a parking lot is primarily concerned with putting threads to sleep and waking them back up; it does not provide the priority-inheritance machinery that made Solaris turnstiles special, as userspace does not need to concern itself with managing priority inheritance, and instead lets the kernel do it through mechanisms like PI-futex on Linux.



## Resources

Turnstiles are well described in the following books:
- *The Design and Implementation of the FreeBSD Operating System*; and
- *Solaris Internals*

Additionally, they were described in a Solaris magazine [here](http://sunsite.uakom.sk/sunworldonline/swol-08-1999/swol-08-insidesolaris.html).



## Footnotes

[^1]: It survives to some extent through [Illumos](https://en.wikipedia.org/wiki/Illumos) 
[^2]: See `turnstile_interlock()`.
[^3]: I actually did manage to do multi-owner PI for a specific case of SMR read-sections, AFAIK this is novel!