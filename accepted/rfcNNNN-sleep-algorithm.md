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

A video walkthrough of the new scheduler is [available on YouTube].

[available on YouTube]: https://youtu.be/HvmQsE5M4cY

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

- **New jobs are available**. These jobs can either come from within
  the threadpool (e.g., by a call to [`join`]) or from outside the
  threadpool. Typically, we are only pushing a single new job, but in
  some cases we may be creating multiple jobs at once.
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

### New jobs are available

The correct response when new jobs are available depends on the state
of the other worker threads:

- If enough threads are currently **idle**, we should probably do
  nothing, as they will likely come and steal the jobs.
- Otherwise, we should try to awaken threads from sleep, so they can
  come and do the work.
  
So you can summarize the correct behavior here as:

- If there are not enough idle threads:
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

- The mechanism to guarantee there is always a thread awake when jobs are pending
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
  decremented. Once it reaches zero, the latch is set. Used for
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

### Tracking the number of 

### Mechanism to judge whether there are idle threads



When injecting new jobs, we want to make a judgement whether there are
enough idle threads -- that is, threads that are actively looking for
work to steal. The idea is that if there are idle threads, we should
avoid waking a thread from sleep, since the idle threads will likely
find the new jobs on their own. Note that typically we have only a
single new job -- but in some cases we may have multiple new jobs at
once.

This RFC proposes using atomic counters to track the number of idle
and sleeping threads (the implementation packs the two counts into a
`AtomicU64`, making it easy to update them together). Whenever a
thread enters the stealing loop, it can increment the "idle thread"
counter. Whenever it exits the stealing loop, it can decrement it.
Similarly, when a thread goes to sleep, it increments the sleeping
counter, and decrements the counter when awoken. (Note that, in this
setup, sleeping threads are considered a subset of idle threads.)

Using a counter has the downside that it means more atomic memory
operations. It has the advantage however of being simple and making it
easy for us to tell how many idle threads are active -- this way, when we
are pushing multiple new jobs, we can tell if the number of new jobs exceeds
the current number of idle threads.

Note that the number of idle/sleeping threads is a moving target, and
thus any attempt to read the counters is always out of date. We
discuss the implications of the race condition here in the [Race
Conditions and Hazards][race-conditions] section below.

### Criteria threads use to decide when to sleep

As part of the busy loop, threads must decide when it is time to stop
sealing and go to sleep. We propose using a simple counter: the thread
will search at most N times, where N is determined by some xperientation.

[race-conditions]: #race-conditions-and-hazards

## Race conditions and hazards

This section documents the various forms of degenerate cases that can
occur, typically as a result of a race condition between a wakeup and
a thread going to sleep, updating a counter, or something like that.

One important change in this system vis-a-vis the existing system is
that we distinguish notifications from *new work* from those that
result from a *latch set*.  We cannot allow the latter to be missed,
as that would result in "livelock" (in this case, a thread sleeping
when it could have made progress).

### Possible sources of missed hazards

In contrast, notifications from new works can be "missed" in a number of
ways in this system:

- If a thread is about to go idle, new work might arrive shortly before,
  which might lead to us waking an additional thread beyond what is needed.
- If a new job arrives just as a thread is about to go to sleep, the
  thread may still be considered idle. We woulld therefore skip waking a thread,
  but the idle thread might go to sleep.
  - This could be obviated by waking workers with the goal that there
    be at least *2* idle threads, but that would cost more CPU time,
    and doesn't completely eliminate the risk (both threads might be
    about to go to sleep).
- If two or more new jobs arrive at close to the same time, and there
  is one idle thread, that thread will only be able to steal one of
  them. The other will wait in a deque until an existing thread
  finishes a job (or creates a new job, which would create a new
  worker).

### Conclusion

Ultimately, the main danger highlighted above is that there could be a
job that gets enqueued and which *could* be stolen if a thread was
awoken, but that thread is never awoken because -- at the time of
posting the job -- it didn't appear necessary.

For workloads with a small number of jobs, it is possible for us to
start fewer threads than the optimal number. If those jobs are cheap,
that's not a big problem, but if they are expensive, it will result in
subpar performance.

If the number of jobs is approximately equal to the number of threads,
this could be a problem. We only get one chance to start a thread per
job. One way to potentially address this: in the code, if we know the
number of new tasks being created ahead of time, we can try to ensure
N threads are awake.

If there are many, fine-grained jobs, then the likelihood of a problem
is much smaller. Each new job offers us a fresh chance to start a new
thread; if there are many jobs, then the existing threads should not
be idle.  Moreover, there are not that many threads available,
relative to the number of jobs, so soon they will all have started.

# Rationale and alternatives

## Potential avenues for future exploration

This section contains some interesting ideas that may be worth
exploring. They mostly represent minor variations on the protocol
described in the RFC.

### Judging idle threads by looking at the local queue

The current "idle threads" tracking uses a global counter. One
interesting alternative when a worker thread is pushing a new job is
to have the thread check its local deque and see whether it is
empty. The idea is that an empty deque likely indicates the presence
of idle threads -- after all, there are enough worker threads to be
stealing whatever work we have produced thus far. The advantage of
this check is that it is local and does not require synchronized
state. The disadvantage is that (a) this check only works from worker
threads, (b) it is less precise, and (c) it cannot estimate how *many*
idle threads there are. It is unclear how big these disadvantages are:
most cases of new job are created by worker threads and consist of a
single job, so (a) and (c) only rarely apply.

This alternative is worth exploring further. In particular, it may
yield advantages of machines with a large number of cores, as
coordinating atomic updates may be more expensive in that environment.

### Waking "blocks" of threads at once

The current system wakes one thread at a time. It might be interesting
to look into waking up 'blocks' of threads at once, in anticipation of
their being new jobs created. For example, we might wake up to the
smallest power of 2 -- so if we already have 5 active threads, we
would start 3 more to bring the total to 8 (some of those might then
fall back asleep). This represents something of a hybrid of the
proposed system and the existing scheduler (described below), in that
the existing Rayon scheduler always woke *all* threads and then
allowed them to fall back asleep.

## Related work

This section contains summaries of some existing systems that were
examined.

### Current behavior

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

### Go scheduler

The go scheduler's approach to this problem is [summarized in this
comment][go]. Based on a superficial read, it appears quite similar:
when a new goroutine is created, if there are no idle threads (called
"spinning"), then a new worker thread (called a "P", for processor) is
awoken (called "unparked").

[go]: https://github.com/golang/go/blob/b25ec50b693eae68de1f020a9566fa14dea47888/src/runtime/proc.go#L31

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
potentially more locks. It may be worth doing some experimentation.

