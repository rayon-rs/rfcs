# Summary
[summary]: #summary

- Introduce a new function, `scope_fifo`, which introduces a Rayon scope
  that executes tasks in **per-thread FIFO** order; in this mode, at
  least with one worker thread, the tasks that are spawned first execute
  first. This is on contrast to the tradition Rayon `scope`, which
  executes in **per-thread LIFO** order, such that tasks that are
  spawned first execute last. Per-thread FIFO requires a small amount of
  indirection to implement but is important for some use-cases.
- Introduce a new function, `spawn_fifo`, that pushes tasks onto the
  implicit global scope. These tasks will also execute in FIFO order,
  in contrast to the [existing `spawn` function][spawn].
- Deprecate the [existing `breadth_first` flag on `ThreadPool`][bf].
  Users should migrate to creating a `scope_fifo` instead, as it is better
  behaved.
  - In the future, the `breadth_first` flag may be converted to a no-op.

[spawn]: https://docs.rs/rayon/1.0.3/rayon/fn.spawn.html
[bf]: https://docs.rs/rayon/1.0.2/rayon/struct.ThreadPoolBuilder.html#method.breadth_first

# Motivation
[motivation]: #motivation

The prioritization order for tasks can sometimes make a big difference
to the overall system efficiency. Currently, Rayon offers only a
single knob for tuning this behavior, in the form of [the
`breadth_first` option][bf] on builds. This knob is not only rather
coarse, it can lead to quite surprising behavior when one intermingles
`scope` and `join` (including stack overflows, see [#590]). The goal
of this RFC is to make more options available to users while ensuring
that these options "compose well" with the overall system.

[#590]: https://github.com/rayon-rs/rayon/issues/590

## Current behavior: Per-thread LIFO

By default, and presuming no stealing occurs, the current behavior of
a Rayon scope is to execute tasks in the reverse of the order that
they are created. Therefore, in the following code, task 2 would
execute first, and then task 1:

```rust
rayon::scope(|scope| {
  scope.spawn(|scope| /* task 1 */ );
  scope.spawn(|scope| /* task 2 */ );
});
```

Thieves, in contrast, steal tasks in the order that they are created,
so a thief would steal Task 1 before Task 2. Once a task is stolen,
any new tasks that it spawns are processed first by the thief, again
in reverse order (but may in turn be stolen by others, again in the
order of creation).

**Implementation notes.** This behavior corresponds very nicely with
the general "work stealing" implementation, where each thread has a
deque of tasks. Each new task is pushed on to the back of the
deque. The thread pops tasks from the back of the deque when executing
locally, but steals from the front of the deque.

## Per-thread FIFO

Unfortunately, for some applications, executing tasks in the reverse
order turns out to be undesirable. One such application is stylo, the
parallel rendering engine used in Firefox. The "basic loop" in Stylo is
a simple tree walk that descends the DOM tree, spawning off a task for each
element:

```rust
fn style_walk<'scope>(
  element: &'scope Element,
  scope: &rayon::Scope<'scope>,
) {
  style(element);
  for child in element {
    scope.spawn(|scope| style_walk(child, scope));
  }
}
```

For efficiency, Stylo employs a per-worker-thread cache, which enables
it to share information between tasks. For this task to be maximally
effective, it is best to process all of the children of a given
element first, before processing its "grandchildren" (this is because
sibling tasks require the same cached information, but grandchildren
may not). However, if we use the default scheduling strategy of
per-thread LIFO, this is not what we get: instead, for a given element
`E`, we would process first its last child `Cn`. Processing `Cn` would
push more tasks onto the thread-local deque (the grandchildren of `E`)
and those would be the next to be processed. In effect, we are getting
a depth-first search when what we wanted was a breadth-first search.

To address this, we currently offer a per threadpool flag called
[`breadth_first`][bf].  This causes us to (globally) process tasks
from the front of the queue first.  This works well for Stylo, but
[interacts poorly with parallel iterators][#590], as previously
mentioned.

Instead of a flag on the threadpool, this RFC proposes to allow users
to select the scheduling when constructing a scope. So stylo would
create its scope using `rayon::scope_fifo` instead of `rayon::scope`:

```rust
rayon::scope_fifo(|scope| {
  ...
});
```

Creating a FIFO scope means that, when a thread goes to process a task
that is part of the scope, it prefers first the tasks that were
created most recently **within the current thread**. If we assume no
stealing, then all tasks are created by one thread, and hence this is
simply a FIFO ordering.

However, when stealing occurs, the ordering can get more complex and
does not abide by a strict FIFO ordering. To illustrate, imagine that
a thread T1 creates a scope S and creates three tasks, A, B and C
within the scope. This thread T1 will begin by executing A, as it is
the task created first. Let us imagine that processing A takes a very
long time, and all the rest of the work proceeds before it completes.

Next, imagine that a thief thread T2 steals the task B. In executing
B, it creates two more tasks, D and E. Once T2 finishes with B, it
will proceed to execute D and E, in that order (presuming they are not
stolen first). Only when it completes E will it go back to T1 and
steal the task C. So the final ordering of *task creation* is A, B, C,
D, E, but the tasks *begin* execution in a different order: A, B, D,
E, C. This order is influenced by what gets stolen and when.

As it happens, this "per-thread FIFO" behavior is a very good fit for
Stylo. It enables each worker thread to keep a cache in its
thread-local data. When T2 steals the task B, its local cache is
primed to process B's children, i.e., D and E. If T2 were to go and
process C now, it would not benefit at all from the cache built up
when processing B.

In fact, similar logic likely applies to many other applications: if
nothing else, the caches on the CPU itself contain the state accessed
from B, and it is likely that the tasks spawned by B are going to
re-use more of those cache lines than task C. (In general, this is why
we prefer a LIFO behavior, as it offers similar benefits.)

# Guide-level explanation

## New functions and types

We extend the rayon-core API to include two new functions, as well as two
corresponding methods on the `ThreadPool` struct:

- `scope_fifo(..)` and `ThreadPool::scope_fifo`
- `spawn_fifo(..)` and `ThreadPool::spawn_fifo`

These two functions (and methods) are analogous to the existing
[`scope`] and [`spawn`] functions respectively, except that they
ensure **per-thread FIFO** ordering.

The `scope_fifo` function (and method) takes a closure implementing
`FnOnce(&ScopeFifo<'scope>)` as argument. The `ScopeFifo` struct (also
introduced by this RFC) is analogous to existing [`Scope`] struct --
it permits one to spawn new tasks that will execute before the
`scope_fifo` function returns. It will offer one method,
`ScopeFifo::spawn_fifo`, that permits one to spawn a (FIFO) task into
the scope, analogous to [`Scope::spawn`].

[`scope`]: https://docs.rs/rayon/1.0.3/rayon/fn.scope.html
[scope_method]: https://docs.rs/rayon/1.0.3/rayon/struct.ThreadPool.html#method.scope
[`spawn`]: https://docs.rs/rayon/1.0.3/rayon/fn.spawn.html
[`Scope`]: https://docs.rs/rayon/1.0.3/rayon/struct.Scope.html
[`Scope::spawn`]: https://docs.rs/rayon/1.0.3/rayon/struct.Scope.html#method.spawn

## Deprecations

The `breadth_first` flag on thread-pools is **deprecated** but (for
now) retains its current behavior. In some future rayon-core release,
it may become a no-op, so users are encouraged to migrate to use
`scope_fifo` or `spawn_fifo` instead.

# Implementation notes

## Implementing `scope_fifo`

Rayon's core thread pool operates on the traditional work-stealing
mechanism, where each worker thread has its own deque. New tasks are
pushed onto the back of the deque, and to obtain a local task, the
thread pops from the back of the deque. When stealing, jobs are taken
from the front of the deque. So how can we extend this to accommodate
the new scheduling modes? 

In addition, we have the goal that nested scopes using these modes
should "compose" nicely with one another (and with the `join`
operation)[^global]. So, for example, if we have a thread nesting scopes like:

[^global]: The existing "global FIFO mode" fails miserably on this criteria, which is a partial motivator for this RFC.

- a per-thread LIFO scope S1 that contains a
  - per-thread FIFO scope S2 that contains
    - a join(A, B) of tasks A and B
    
then we should execute:

- first, the tasks from the join (in reverse order, so A and then B)
- then, the tasks from S2, in the order that they were created
- then, the tasks from S1, in the reverse order from which they were created.

Implementing **Per-thread LIFO** scheduling is therefore very
easy. Each new job pushed onto the stack is simply pushed directly
onto the back of the deque. Worker threads pop local tasks from the
back of the deque but steal from the front (if the `breadth_first`
flag is true, then worker threads pop local tasks from the front as
well).

Implementing **Per-thread FIFO** requires a certain amount of
indirection. The scope creates N FIFOs, one per worker thread (as of
this writing, the number of worker threads is fixed; if we later add
the ability to grow or shrink the thread-pool dynamically, that will
make this implementation somewhat more complex). When a new task is
pushed onto a FIFO scope by the worker with index W, we actually push two items:

- First, we push the task itself onto the FIFO with index W.
  This task contains the closure that needs to execute.
- Second, we push an "indirect" task onto the worker's thread-local
  deque. This task contains a reference to the FIFO for the worker
  index W that created it, but does not record the actual closure that
  needs to execute.

Like any other task, this "indirect task" may be popped locally or
stolen. In either case, when it executes, it must first find the
closure to execute. To do that, it will find the FIFO for the worker W
that created it and pop the next closure from the front, which it can
then execute. Any new tasks pushed by this task T will be pushed onto
the *current* thread, which is not necessarily the thread W that
created it.

Note that the FIFO mode does impose some "indirection overhead"
relative to the LIFO execution mode. This is partly what motivates the
default, as well backwards compatibility concerns. In any case, the
overhead does not seem to be too large: the [prototype
implementation][] of these ideas was [evaluated experimentally by
@bholley][experiment], who found that it performs equivalently to
today's code.

[prototype implementation]: https://github.com/rayon-rs/rayon/pull/601#issuecomment-433242023
[experiment]: https://github.com/rayon-rs/rayon/pull/601#issuecomment-433242023

## Implementing `spawn_fifo`

The traditional `spawn` function used by Rayon behaves (roughly) "as
if" there were a global scope surrounding the worker thread: presuming
that spawn is executed from inside a worker thread, it simply pushes
the task onto the current thread-local deque. (When executed from
**outside** a worker thread, the task is added to the "global
injector"; it will eventually be picked up by some worker thread and
executed.)

`spawn_fifo` can be implemented in an analogous way to `scope_fifo` by
having each worker thread have a global FIFO, analogous to the FIFOs
created in each `scope_fifo`. `spawn_fifo` then pushes the true task
onto this FIFO as well as an "indirect task" onto the thread-local
deque, exactly as described above.

# Rationale and alternatives

**Why change at all?** Two things are clear:

- There is a need for FIFO-oriented scheduling, to accommodate Stylo.
- The existing, global implementation -- while simple -- has serious
  drawbacks. It works well for Stylo, but it doesn't "compose" well
  with other code, that may wish to use parallel iterators or
  join-based invocations.
  
These two facts together motivate moving to something like the
per-thread FIFO behavior described in this RFC.

**Why offer both FIFO and LIFO modes?** A serious alternative however
would be to offer **only** this behavior, and not support per-thread
LIFO -- this is the design which e.g. [@stjepang is advocating
for][stjepang-1]. In general, Rayon prefers offering fewer knobs, so
that would be a general fit. However, there are some advantages to
offering more scheduling modes:

[stjepang-1]: https://github.com/rayon-rs/rfcs/pull/1#issuecomment-437074748

- Per-thread LIFO is the **current default behavior** of scopes. This
  is "semi-documented". While not strictly part of our semver
  guarantees, altering this behavior could negatively affect existing
  applications.
- Per-thread LIFO offers the **most efficient** implementation in a
  "micro" sense, as it can build directly on the work-stealing deques
  and does not require any indirection. It also has desirable cache
  behavior, as threads will tend to continue the "most recent thing"
  you were doing, which is also the thing where the caches are
  warmest, etc. **These two cases together argue that, for cases where
  the application doesn't otherwise care about execution order,
  per-thread LIFO is a good choice.** This also applies to
  applications that wish to choose another scheduling constraint (see
  the next section).

**Should we offer additional scheduling modes?** Another question
worth considering is "why stop with these two modes"?  For example,
the [Rayon demo application for solving the Travelling Salesman
Problem][tsp] also [uses the "indirect task"
trick][tsp-indirect]. However, in that case, it uses a (locked)
priority queue to control the ordering. Similarly, earlier versions of
this RFC considered a "global FIFO" ordering where tasks always
executed in the order they were pushed, regardless of whether they
were stolen or not. Rayon could conceivably offer some more flexible,
priority-queue like mechanism as well. However, it's not clear that
this is worth doing, since one can always achieve the same effect in
"user space", as the TSP application does. (For this purpose,
per-thread LIFO tasks are a great fit, as they add minimal overhead.)

[tsp]: https://github.com/rayon-rs/rayon/blob/a68b05ce524f79d7e7a5065714a8d3ca40ce8d4b/rayon-demo/src/tsp/
[tsp-indirect]: https://github.com/rayon-rs/rayon/blob/a68b05ce524f79d7e7a5065714a8d3ca40ce8d4b/rayon-demo/src/tsp/step.rs#L50-L51

**Should `scope_fifo` return a [`Scope`] and not a `ScopeFifo`?** The
[prototype implementation] shares the same `Scope` type for both
`scope_fifo` and `scope`, and stores a boolean value to remember which
"mode" is in use. This RFC proposes a distinct return type, which
gives us the freedom to avoid using a dynamic boolean at runtime
(though the implementation is not required to take advantage of that
freedom).

**What to do with the `breadth_first` flag?** Earlier drafts of this
RFC proposed making the `breadth_first` flag a no-op immediately. It
was decided however to simply deprecate the flag but keep its current
behavior for the time being: users of `breadth_first` are encouraged
to migrate to `scope_fifo`, however, since the `breadth_first` flag
may become a no-op in the future (this would simplify the overall
rayon-core implementation somewhat).

# Unresolved questions

None.
