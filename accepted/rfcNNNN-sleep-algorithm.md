# Summary
[summary]: #summary

An improved approach to sleeping worker threads in Rayon.

# Motivation
[motivation]: #motivation

Rayon's existing approach to putting threads to sleep can lead to
excessive CPU usage (see e.g. [rayon-rs/rayon#642]). In the current
algorithm, threads can gradually put themselves to sleep if they don't
find work to do.  They do this one at a time. But as soon as *any*
work arrives (or -- in fact -- even any work *completes*) all threads
awaken.

[rayon-rs/rayon#642]: https://github.com/rayon-rs/rayon/issues/642

This RFC proposes an alternative algorithm, and explores some of the
design options and tradeoffs available. It does not claim to be
exhaustive and feedback is most certainly desired on alternative
approaches!

# Guide-level explanation

[how-rayon-works]: #how-rayon-works-at-a-high-level

## How Rayon works at a high level

Because the sleep behavior is pervasive, explaining a new algorithm
requires giving a high-level overview of how Rayon's worker threads
work.

### Latches

A core concept in Rayon is the *latch*. A [latch] is a signaling
mechanism used to indicate when an event has occurred. The latch
begins as *unset* or *false*. It can be *set* by any thread. Once it
is set, it can never be unset.

Each latch is associated with one *owner thread*. This is the thread
that may be blocking, waiting for the latch to complete. The fact that
there is exactly one well-known owner thread is key to our wakeup
design.

An example of where a latch is used is the function [`join`].
[`join`] must block until both closures have terminated, but one of
those closures may be executed by another thread (actually, both, if
[`join`] is invoked from outside the thread pool). Therefore, [`join`]
creates a latch for each parallelizable closure. This latch will be
set once the closure has executed. In the meantime, the thread that
called [`join`] can block on the latch.

[worker-thread-loop]: #worker-thread-loop

### Worker thread loop

The core worker-thread "busy loop" works as follows. First, it is
invoked with some latch L that it is waiting on. Initially, this is
the "terminate latch", which is used to signal when the thread-pool as
a whole ought to terminate. Later on, it is the latch associated with
a particular job.

- While the latch L is not set:
  - Try to steal a job from other threads
  - If a job is found, execute that job
  - Otherwise, consider going to sleep
  
The process for **executing a new job** is simply to invoke the
closure for the job.  This job may in turn execute [`join`] or
[`scope`], which would create a fresh latch L2 and return us to the
busy loop, but blocking on L2.
  
The process for **considering going to sleep** is of course what we
are defining in this document and is covered in future sections.
Naturally, if a thread does go to sleep, it will wish to be awoken
when either new work is available to be stolen or the latch becomes
set.

## Causes for events

To start, let's list the various reasons that a Rayon thread may need
to wake from sleep:

- **A new job is available**. This job can either come from within the
  threadpool (e.g., by a call to [`join`]) or from outside the
  threadpool.
- **A [latch] has been set**. [Latches][latch] are the signaling
  mechanism used in Rayon to indicate when a job has completed. So,
  for example, a call to [`join`] cannot return until both closures
  have finished, and latches are used to block until that has
  occurred.
- **The thread-pool is terminating.** When a Rayon [`ThreadPool`] is
  dropped, it signals its threads so they too can terminate.

[latch]: https://github.com/rayon-rs/rayon/blob/master/rayon-core/src/latch.rs#L7
[`join`]: https://docs.rs/rayon/1.2.0/rayon/fn.join.html
[`ThreadPool`]: https://docs.rs/rayon/1.2.0/rayon/struct.ThreadPool.html

## Handling wakeups for each event

Let's look at each event in turn and see what kind of wakeup is most
appropriate. Note that Rayon uses the playful term "tickling" for the
act of waking a thread.

### A new job is available

The correct response when a new job is available depends on the state
of the other worker threads:

- If any threads are currently **idle**, we should probably do nothing,
  as they will likely come and steal the job.
- If all other threads are either **sleeping** or **busy**, then we
  should try to awaken another thread (presuming that there are any
  still available).
  
So you can summarize the correct behavior here as:

- If there are no idle threads:
  - If there are sleeping threads:
    - Pick one sleeping thread and wake it.

We'll explore later some of the options for tracking whether there are
idle threads.

[latch-set]: #a-latch-has-been-set

### A latch has been set

As noted [above][how-rayon-works], latches are used so that the thread
which invoked a method like [`join`] can tell when its closures have
terminated. That thread will be somewhere in its [worker thread
loop][worker-thread-loop]: 

- While the latch L is not set:
  - Try to steal a job from other threads
  - If a job is found, execute that job
  - Otherwise, consider going to sleep

The correct behavior when a latch is set depends then on where in that
process the thread currently is. *If* the thread has gone to sleep --
and specifically gone to sleep as part of *this* loop and not some
other loop -- then it makes sense to wake it up.

In the other other cases, it doesn't make sense to wake the thread up:

- If the thread is still trying to steal jobs, then it is not asleep.
- If the thread has found another job and started to execute it, it
  could have gone to sleep as part of that process. But in that case
  it is currently blocked on *some other latch*, so if we wake it up,
  it will simply waste CPU time spinning for jobs to steal and
  eventually go back to sleep.

So the correct behavior here is:

- If the owner thread of this latch is blocked on this latch:
  - Wake the owner thread.
  
We'll explore later how we can track if owner thread is blocked on th
latch.

### The thread-pool is terminating

This event can be handled by creating one latch per worker thread at
startup and then, when the user has requested termination, signaling
each latch in turn. Therefore, it is equivalent to the mechanism
described in [the previous section][latch-set].

# Implementation notes

The previous section summarized at a high-level the way that the
threads work and the various wake-up scenarios we need to handle.
This section goes into more detail on how the implementation will work.

## Defining key mechanisms

This section aims to give details on the following points:

- The mechanism for threads to go to sleep and be awoken.
- The mechanism for latches to know when a thread is sleeping on them.
- The mechanism to judge whether there exist idle threads.
- The criteria threads use in their busy loop to decide when to go to
  sleep if they have not found work.

### Mechanism for threads to go to sleep and be awoken

For this we will simply use an array of locks and condition variables,
with one `WorkerSleepState` per worker thread:

```rust
struct WorkerSleepState {
  is_sleeping: Mutex<bool>,
  condvar: Condvar,
}
```

The `is_sleeping` lock guards a boolean that indicates whether the
thread is presently asleep: this value is only ever mutated by the
thread itself, which sets the bolean to true before going to sleep and
to false when it wakes up.

Other threads can thus awaken the worker by grabbing the lock and
invoking `notify_one` on the `condvar` (or `notify_all`, it makes no
difference).

### Mechanism for latches to know when a thread is sleeping on them

There are two kinds of latches that are relevant here:

- [`SpinLatch`] -- awaits a single `set`, which causes it become, well, set. Used for [`join`].
- [`CountLatch`] -- contains a counter that can be incremented and
  decrementd. Once it reaches zero, the latch is set. Used for
  [`scope`], where the counter tracks the number of active, spawned
  tasks.

We can actually factor out a "core" latch type that will be used by
both them, creatively called here `CoreLatch`. Like `SpinLatch`, this
`CoreLatch` type awaits a single set. In the case of `CountLatch`, the
`CoreLatch` will be guarded by a counter and will be set only once
that counter reaches zero.

There are three core latch operations:

- **Probe**: executed only by the owner thread T, this simply checks if the latch is
  set. It is done as part of the busy loop and just involves reading the field.
- **Sleep**: executed only by the owner thread T, this causes the owner to go to sleep
  until the latch is set (or the owner thread is otherwise awoken).
- **Set**: executed by any thread. This must wake the owner thread T if they are sleeping.

The precise details of how these methods are implemented is not
especially important. One possible implementation is covered in
[Appendix A][appendix-a].

### Mechanism to judge whether there are idle threads

When injecting new work, we want to make a judgement whether there are
idle threads -- that is, threads that are actively looking for work to
steal. The idea is that if there are idle threads, we should avoid
waking a thread from sleep, since the idle thread will likely find the
work on its own. This itself is obviously something of a heuristic:

- we don't know if there have been other work items pushed that the idle thread 
  has yet to find;
- we don't know if the idle thread is *about* to go to sleep.

Later in the RFC, we'll discuss some of the ways we can tweak the
proposed protocol to try and account for these scenarios.

This section discusses two options for tracking whether there are idle
threads.

**Option A: The local-queue heuristic.**

This RFC proposes a simple heuristic for judging whether there are
idle threads: when a worker thread is about to push a new job onto its
stack, it simply checks whether its local stack is presently empty. If
not, that can be taken as a signal that all the other threads are
busy, because they haven't found the time yet to steal our jobs. In
contract, if the local stack is empty, it suggests that there *are*
idle threads around.

Obviously, this is just a heuristic. For example, the first push that
a thread does will always judge other threads to be idle. To help with
this scenario, we can augment the check by also tracking the total
number of non-sleeping threads (using a global, atomic counter).

Therefore, the check for a worker when pushing work can be:

- If all other threads are sleeping: wake one of them.
- Else, if my local stack is non-empty: wake one of them.

NB. Rayon uses the [crossbeam-deque] package internally. Importantly,
it offers the [`is_empty`] method for its queues.

[crossbeam-deque]: https://crates.io/crates/crossbeam-deque
[`is_empty`]: https://docs.rs/crossbeam-deque/0.7.1/crossbeam_deque/struct.Worker.html#method.is_empty

**Option B: Track idle threads (more) precisely.**

The obvious alternative is to introduce two counters: the number of
sleeping threads *and* the number of idle threads. Whenever a thread
enters the stealing loop, it can increment the "idle thread"
counter. Whenever it exits the stealing loop, it can decrement it. 
The plus side is that we can get a more reliable read on whether there
are idle threads. To decide if there are idle threads, we can simply
read the counter and check whether it is non-zero.

The downside of this approach is that it involves more atomic
increments. One particularly bad scenario is when a stealer thread
steals a "leaf job" -- that is, a job that doesn't spawn any further
jobs. This means that we'll decrement the "idle thread" counter upon
stealing and then re-increment shortly thereafter.

It should be noted that this tracking is *still* imprecise. As noted
earlier, this is still ultimately a heuristic: there are still races
where the thread might have concluded that it is time to go to sleep,
or the thread might be about to find another job.

**Which option?**

This should be decided with experimentation. Both options are fairly
easy to implement.

### Criteria threads use to decide when to sleep

As part of the busy loop, threads must decide when it is time to stop
sealing and go to sleep. We propose using a simple counter: the thread
will search at most N times, where N is determined by some xperientation.

## Tuning opportunities

There are a number of opportunities to tweak this protocol:

- How many threads do we wake when new work arrives and we judge that
  there are no idle threads? Instead of only waking 1 thread, it might make
  sense to jump to a "threshold" point.
- How 

# Rationale and alternatives

## Current behavior

The current behavior differs in three respects from what is described
in this document. A detailed look is available in [the README from the
source][sleep-README].

[sleep-README]: https://github.com/rayon-rs/rayon/blob/68edcf6d12301cfdbe2a72fefada21fd45052891/rayon-core/src/sleep/README.md

First, the current sleep module always wakes **all** threads at once
when a new event arrives. As such, it requries only a single mutex and
a single condvar. This can be good: it ensures that threads are
available to work as quickly as possible. But it also leads to high
CPU usage.

Once threads are awake, they can go to sleep individually, and so it
is theoretically possible for Rayon to run at under full capacity. In
practice, however, this is limited by the second difference: the
current sleep module does not track which thread is waiting on what
latch. As such, it must generate events to wake all threads each time
a job is completed, and not only when new work is available. This is
because completing a job may have released a latch that a sleeping
thread was blocked on.

Finally, the current sleep module has a more complex protocol for
*going* to sleep. The complete sleep state is coordinated using a single
`AtomicUsize`. In order for threads to go to sleep, they first register
themselves in this usize as "the sleepy worker", and then do a bit more
searching to steal work. Only once those searches have failed do they
go to sleep. This means, among other things, that threads can only go
to sleep one at a time.

The advantage of this "sleep one at a time" system is that if new work
*does* arrive in this period, a single atomic swap can be used to
notify the sleepy worker and keep them awake.

# Unresolved questions

- Whether to use precise idle thread tracking or the local-queue heuristic.
- See the "tuning opportunities" discussed above.

[appendix-a]: #appendix-a-core-latch-protocol

# Appendix A: Core latch protocol

The protocol for a core latch can be done with a single counter with
four values:

- UNSET
- SLEEPY
- SLEEPING
- SET

The latch is initially UNSET. 

**Implementing the sleep operation.** When a busy loop for latch L
wishes to go to sleep, it performs the following actions. These
actions operate both on the flag for the latch L and on the
lock/condvar for the thread itself:

- Compare-and-exchange the latch flag from CLEAR to SLEEPY
  - If this operation fails, then the flag must have been SET by someone else,
    so we can abort because there is no longer a reason to block.
- Acquire the sleep lock for the thread (this is not part of the latch)
- Compare-and-exchange latch flag from SLEEPY to SLEEPING
  - If this operation fails, then the flag must have been SET by someone else,
    so we can abort because there is no longer a reason to block.
- Set the "is sleeping" flag for the thread to true (this value is protected by the lock)
- Block on the condvar
- When we wake, set the "is sleeping" flag for the thread to false
- Compare-and-exchange latch flag from SLEEPING to UNSET
  - If this operation fails, then the flag must have been SET while we were sleeping,
    which is fine.

One subtle point: we do not loop in this algorithm at any point. This
is somewhat atypical, as the usual pattern for blocking on a condvar
is to loop until some condition is met. In our case, that loop is
exterior to the sleep operation (it is the "worker thread loop",
effectively). It is theoretically possible for condvars to spuriously
wake up our thread, but we cannot distinguish that from a wakeup
because there is now work available to steal.

**Implementing the set operation.** The set operation can be implemented
quite simply. It requires knowing the owner thread T.

- Unconditionally swap the latch flag with SET
- Examine the previous value:
  - UNSET -- no action is needed, no thread was sleeping; return
  - SLEEPY -- no action is needed, the thread was about to sleep but had not yet,
    and now it will not (because it will attempt a compare-and-exchange and fail); return
  - SLEEPING -- continue
  - SET -- no action is needed, latch was already set; return
- Acquire the sleep lock for the thread T and notify the condvar
  - It is important to acquire the lock first: this ensures that that the thread
    has actually gone to sleep
    
**Notes.** First, this protocol is simply what I came up with when
thinking about the problem. I'm sure I'm not the first to think of it,
and it undoubtedly has a name. There are also obviously variations of
this protocol. For example, the SLEEPY state could be eliminated,
which would yield fewer compare-and-exchange operations but
potentially more locks. It may be worth doing some experimentation

