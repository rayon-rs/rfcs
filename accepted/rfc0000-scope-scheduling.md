# Summary
[summary]: #summary

Offer more variations on the `scope` construct that give more control
over the relative scheduling of spawned tasks. The choices would be:

- **Default:** it is not important which tasks get scheduled when. Let
  Rayon pick based on which can be most efficiently implemented. For
  now, this is "per-thread LIFO", but it could change in the future.
- **Per-thread LIFO:** The task that the current thread spawned most
  recently will be the first to be executed. Thieves will steal the
  thread that was spawned **first** (and thus would be the **last**
  one that the current thread would execute). Tasks spawned by stolen
  tasks will be processed first by the thief, again in a LIFO order,
  but maybe stolen by other threads in turn. This is the current
  default and can be implemented with the highest "micro efficiency".
- **Per-thread FIFO:** The task that the current thread spawned first
  is executed first. Thieves will also steal tasks in the same
  order. Tasks spawned by stolen tasks will be processed first by the
  thief. This is "roughly" the behavior today for thread-pools that
  enabled the `breadth_first` scheduling option.
- **Global FIFO:** Tasks always execute in the order in which they
  were spawned, regardless of whether they were stolen or not.

# Motivation
[motivation]: #motivation

The prioritization order for tasks can sometimes make a big difference
to the overall system efficiency. Currently, Rayon offers only a
single knob for tuning this behavior, in the form of [the
`breadth_first` option][bf] on builds. This knob is not only rather
coarse, it can lead turns out to have quite surprising behavior when
one intermingles `scope` and `join` (including stack overflows, see
[#590]). The goal of this RFC is to make more options available to
users while ensuring that these options "compose well" with the
overall system.

[#590]: https://github.com/rayon-rs/rayon/issues/590
[bf]: https://docs.rs/rayon/1.0.2/rayon/struct.ThreadPoolBuilder.html#method.breadth_first

## Current behavior: Per-thread LIFO

By deafult, and presuming no stealing occurs, the current behavior of
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
locally, but steals from the front of the dequeue.

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
create its scope not via `rayon::scope` but rather using the
`ScopeBuilder`:

```rust
rayon::ScopeBuilder::per_thread_fifo().create(|scope| {
  ...
});
```

Creating a per-thread FIFO scope means that, when a thread goes to
process a task that is part of the scope, it prefers first the tasks
that were created most recently **within the current thread**. If we
assume no stealing, then all tasks are created by one thread, and
hence this is simply a FIFO ordering.

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

## Global FIFO

Both the per-thread LIFO and per-thread FIFO orderings are somewhat
dynamic and unpredictable. For some applications, it may be desirable
to have tasks that execute in a fixed order. For this reason, we offer
another option: global FIFO. In this ordering, tasks always begin
execution in the same order that they were created.

# Guide-level explanation

We extend the rayon-core API to include a `ScopeBuilder`. Presently,
the `ScopeBuilder` will offer three "construction" methods, used to
select the mode:

```rust
pub struct ScopeBuilder {..}

impl ScopeBuilder {
  fn per_thread_lifo() -> Self { }
  fn per_thread_fifo() -> Self { }
  fn global_fifo() -> Self { }
}
```

Finally, the `ScopeBuilder` offers a method to construct the scope:

```rust
impl ScopeBuilder {
  pub fn create<'scope, OP, R>(op: OP) -> R
  where
    OP: for<'s> FnOnce(&'s Scope<'scope>) -> R + 'scope + Send,
    R: Send,
  {
    ..
  }
}
```

The existing `rayon::scope` function is then equivalent to

```rust
rayon::ScopeBuilder::per_thread_fifo().create(|scope| {
  ..
})
```

# Implementation notes

## Controlling ordering

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

- first, the tasks from the join (in reverse order, so B and then A)
- then, the tasks from S2, in the order that they were created
- then, the tasks from S1, in the reverse order from which they were created.

Implementing **Per-thread LIFO** scheduling is therefore very
easy. Each new job pushed onto the stack is simply pushed directly
onto the deque.

Implementing **Global FIFO** scheduling is a bit more complex but
still fairly simple. The scope has an associated parallel FIFO. When a
new task is spawned, we push the task onto this FIFO. We then **also**
push an "indirect" task onto our thread-local deque.  Unlike in the
per-thread LIFO case, this indirect task doesn't correspond to a
specific task pushed by the user: it is merely a "token", indicating
that there is work to be done. Like any other task, this "indirect
task" may be popped locally or stolen. In either case, when it
executes, the indirect task will first pop from the front of the
scope's FIFO to obtain its actual job. It will then execute the task
it finds there.

Implementing **Per-thread FIFO** is similar. The scope creates N
FIFOs, one per worker thread (as of this writing, the number of worker
threads is fixed; if we later add the ability to grow or shrink the
thread-pool dynamically, that will make this implementation somewhat
more complex). When a new task is created in the worker thread with
index X, it will be pushed onto the FIFO with index X. The indirect
task also records this index X. When the "indirect task" is executed,
it pops a task T from the FIFO X and executes it. But note that any
new tasks pushed by this task T will be pushed onto the *current*
thread, which is not necessarily the thread X.

Note that both FIFO modes do impose some "indirection overhead"
relative to the LIFO execution mode. This is partly what motivates the
default, as well backwards compatibility concerns. In any case, the
overhead does not seem to be too large: the [prototype
implementation][] of these ideas was [evaluated experimentally by
@bholley][experiment], who found that it performs equivalently to
today's code.

[prototype implementation]: https://github.com/rayon-rs/rayon/pull/601#issuecomment-433242023
[experiment]: https://github.com/rayon-rs/rayon/pull/601#issuecomment-433242023

# Rationale and alternatives

**Why change at all?** Two things are clear:

- There is a need for FIFO-oriented scheduling, to accommodate Stylo.
- The existing, global implementation -- while simple -- has serious
  drawbacks. It works well for Stylo, but it doesn't "compose" well
  with other code, that may wish to use parallel iterators or
  join-based invocations.
  
These two facts together motivate moving to something like the
per-thread FIFO behavior described in this RFC.

**Do we need multiple scheduling modes for scopes?** A serious
alternative however would be to offer **only** this behavior, and not
support per-thread LIFO nor global FIFO -- this is the design which
e.g. [@stjepang is advocating for][stjepang-1]. In general, Rayon
prefers offering fewer knobs, so that would be a general fit. However,
there are some advantages to offering more scheduling modes:

[stjepang-1]: https://github.com/rayon-rs/rfcs/pull/1#issuecomment-437074748

- Per-thread LIFO is the **current default behavior** of scopes. This
  is "semi-documented". While not strictly part of our semver
  guarantees, altering this behavior could negatively affect existing
  applications.
- Per-thread LIFO offers the **most efficient** implementation in a
  "micro" sense, as it can build directly on the work-stealing deques
  and does not require any indirection. It also has desirable cache
  behavior, as threasd will tend to continue the "most recent thing"
  you were doing, which is also the thing where the caches are
  warmest, etc. **These two cases together argue that, for cases where
  the application doesn't otherwise care about execution order,
  per-thread LIFO is a good choice.** This also applies to
  applications that wish to choose another scheduling constraint (see
  the next section).
- Global FIFO, meanwhile, is the only scheduling order of the 3 that is
  independent from the work-stealing behavior.

**Should we offer additional scheduling modes?** Another question
worth considering is "why stop with these three modes"?  For example,
the [Rayon demo application for solving the Travelling Salesman
Problem][tsp] also [uses the "indirect task"
trick][tsp-indirect]. However, in that case, it uses a (locked)
priority queue to control the ordering. Rayon could conceivably offer
some more flexible, priority-queue like mechanism as well. However,
it's not clear that this is worth doing, since one can always achieve
the same effect in "user space", as the TSP application does. (For
this purpose, per-thread LIFO tasks are a great fit, as they add
minimal overhead.)

[tsp]: https://github.com/rayon-rs/rayon/blob/a68b05ce524f79d7e7a5065714a8d3ca40ce8d4b/rayon-demo/src/tsp/
[tsp-indirect]: https://github.com/rayon-rs/rayon/blob/a68b05ce524f79d7e7a5065714a8d3ca40ce8d4b/rayon-demo/src/tsp/step.rs#L50-L51

**Is a builder really necessary?** We could simply add some more
variants of `rayon::scope`, such as
`rayon::scope_per_thread_fifo`. Perhaps that would be easier. We can
always move to a builder at some later point.

**Should we encode the scheduling mode in the type of the scope?** The
prototype implementation of `Scope` stores the scheduling mode as an
"enum". An alternative would be to use generics to encode the mode;
this would avoid a dynamic `if` to select between per-thread
LIFO/per-thread FIFO etc, but at the cost of more complexity in the
user-visible types (it would also mean that we must expose more than
one scope type, as the existing -- stable -- rayon-core API does not
contain a generic). Given the performance measurements, it is unlikely
that the more complex types are worthwhile. This change could also be
made at some later time, conceivably, though it would require more
additions to the `ScopeBuilder` API.

# Unresolved questions

None.
