+++
title = "Deadlock Detection Made Easy"
date = 2026-11-17

[extra]
type = "Post"
+++

## Introduction
I recently implemented a deadlock detection algorithm in [zag](/zag) which turned out quite simple,
and I thought I would describe it here. While deadlocks are fairly easy to avoid by taking care of following statically-defined lock orderings, 
there are often cases unaccounted for, especially in a kernel, where there are many layers of code and asynchronous events 
that could take locks.

## The Goal
Our goal is to fundamentally detect cycles in a graph where the nodes are held lock *classes* and the edges represent
an ancestor -> descendant relationship; if an ordering `A -> B` is defined inside the system, then we must ensure that
no relationship `B -> A` ever occurs in the graph, which would otherwise cause a possible cycle `A -> B -> A`.
Transitive dependencies must also be kept track of, e.g in the case of `A -> B -> C`, `B -> C` and `A -> C` must both be enforced.

I speak of lock *classes* here because this tracking is performed on a class/lock kind granularity, and not on a per-physical-lock basis. 
For instance, there is physically one lock per thread protecting its state, but the ordering is enforced on its kind,
which is "thread lock".

{{<figure src="/lockdep1.svg" alt="" caption="Graph with a cycle in it" width="10%"/>}}

## The Algorithm
One way this can be solved is by using an *adjacency matrix*, 
where all classes have slots in a matrix `M` such that `M[A][B]` is true if `B` is reachable through `A`.
To handle transitive lock dependencies, all the nodes reachable by a child of `A` must also be appended to `A`.

Note that lock orderings do not need to be statically encoded by the programmer, they can be
discovered dynamically at runtime; while it is *nice* to do so to ensure that the code does as intended, it is not strictly necessary
for the correctness of the algorithm.


It is unclear what to do when two locks of the same classes are held at the same time, and it is ultimately up to the implementation to decide. 
In my implementation, I decided not to error on this since it is assumed that holding two locks of the same class is done with care, and such code is kept cycle-free by a using broader, undetectable, ordering defined by the programmer, 
for instance, by using the virtual address of both locks to define the order in which to take them. 

## The Implementation
The adjacency matrix can be implemented efficiently by having an array containing `N` bitmaps where `N` is the number of 
maximum lock classes in the system, and where each bit in the bitmap represents a class.

When an edge `A -> B` is formed, the B-th bit of `M[A]` is set to 1.
When a subsequent edge `B -> A` attempts to be formed as the result of an ordering inversion, the bit `M[A][B]` being set 
is noticed and runtime panic occurs.

The handling of transitive lock dependencies can be done simply by merging two bitmaps using bitwise or.
When `A -> B` is formed, `M[A] |= M[B]` ensures that all the descendents of B are also marked as descendants of A.


{{<figure src="/lockdep2.svg" alt="" caption="Graph with reachability bitmaps" width="30%"/>}}


Edges are formed when a lock is held; held locks are kept track of on a per-thread or per-CPU basis (in the case of spinlocks), this can be done by adapting the lock acquiring and releasing functions themselves.

All lock classes are registered in a hash table mapping the class's name to its index in the matrix; registration is performed the first time a lock class is initialized.


In zag, I tentatively called this feature *warden*, and the feature is used at lock state-modifying points like so:

```zig
pub const Mutex = struct {
    /// The thread currently holding the mutex, null if unlocked.
    state: std.atomic.Value(State),
    wd: kep.warden.LockData = undefined,

    pub fn init(class: []const u8) Mutex {
        if (config.warden) {
            const cl = kep.warden.find_or_create_lock_class(class);
            return .{
                .state = ...,
                .wd = cl,
            };
        }

        return .{ .state = ... };
    }

    pub fn acquire(self: *Mutex) bool {
        // ...
            if (config.warden) {
                kep.warden.acquired(self, self.wd, .Lock);
            }
        // ...
    }

   pub fn release(self: *Mutex) void {
        // ...
            if (config.warden) {
                kep.warden.released(self, .Lock);
            }
        // ...
    }
```

The `config.warden` check here is done at compile-time and all lock-tracking code is therefore not emitted in builds without
the warden enabled.

A powerful realization is that this scheme can do more than check for cycles, since it hooks into lock functions. 
This could, for example, also be used to check that preemption is disabled when a spinlock is acquired, or check that all spinlocks are subordinates of blocking locks.

I've also implemented a similar scheme in [MINTIA](https://github.com/xrarch/mintia2) and was able to use those lock tracking points to measure contention. 

Debugger support was also added to show off the lock hierarchies in the system:

{{<figure src="/mintia-locks.png" alt="" caption="All the locks in MINTIA viewed in the debugger" width="80%"/>}}

(I admit, this is mostly eye-candy more than anything else!)

## Resources
General idea/algorithms:
- This is somewhat inspired by Linux's feature named `lockdep`, albeit much simpler.
- Most of the algorithm is similar to FreeBSD's WITNESS graphs, but using bitmaps.

Code:
- [FreeBSD WITNESS](https://elixir.bootlin.com/freebsd/v15.1/source/sys/kern/subr_witness.c)
- [zag implementation](https://github.com/rdmsr/zag/blob/3c9c00aa176c508036f057340bb0438157e694e7/src/sys/kern/warden.zig)
- [MINTIA implementation](https://github.com/xrarch/mintia2/blob/main/OS/Executive/Ke/KeLockDep.jkl)
