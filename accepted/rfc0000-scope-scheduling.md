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
to select the scheduling more per-scope. So stylo would create its
scope not via `rayon::scope` but rather using the `ScopeBuilder`:

```rust
rayon::ScopeBuilder::per_thread_fifo().create(|scope| {
});
```

## Global FIFO


So, if the rough stylo model is something like this:


# Guide-level explanation

How does this affect public Rayon APIs? If this is purely an
implementation detail, it's ok to note that it doesn't affect Rayon
users very much.

# Implementation notes

Describe at a high-level details of the implementation. This doesn't
have to be super detailed; e.g., if we are adding a new
`ParallelIterator` API, it it probably fine to leave some of this up
to the Rayon PR. It is also fine to open a PR on the Rayon repository
and link it to this RFC (and vice versa).

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions

Anything left unresolved that we'll need to work out and evaluation
during implementation? Feel free to leave this section blank if you
can't think of anything.  "
