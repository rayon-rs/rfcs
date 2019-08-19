# Summary
[summary]: #summary

This rfc is about merging [rayon-logs](https://github.com/wagnerf42/rayon-logs) into rayon.


# Motivation
[motivation]: #motivation

- It is very clear that we need a performant logging library in order to understand how to fine tune
parallel algorithms. I actually started some research projects on rayon before stopping them and starting
with the development of a logging crate.
- Logs also enable a better teaching of both rayon's use and parallel algorithms
to students.

You can see for yourself on my [different](http://www-id.imag.fr/Laboratoire/Membres/Wagner_Frederic/category/rayon-logs.html)
[webpages](http://www-id.imag.fr/Laboratoire/Membres/Wagner_Frederic/category/rayon-adaptive.html) for nice eye candy.

<div> <img src="http://www-id.imag.fr/Laboratoire/Membres/Wagner_Frederic/images/downgraded_manual_max.svg"/> </div>


Now another question is since `rayon_logs` is available as an external crate, why even change rayon's code ?

`rayon-logs` is nice but it cannot log everything. Currently only `join` and `scope` are logged and not on the global pool.
There is a hack for iterators but it is hacky at best since it cannot for example log a parallel collect.

In fact all parallel algorithms in rayon itself like sorts and collects would need to be copy-pasted in `rayon-logs`
(already the case for the merge sort as a demonstrator).
Copy pasting is tedious and unmaintainable.

On top of that it requires some complex use of `cfg()` and features especially when building crates on crates which use rayon because
I need to re-export `rayon-logs`'s `join` and `scope` everywhere.

Having the logs inside rayon itself would enable to leave user code almost unchanged and just re-compile with `--features=logs`
to get a code animation.

Gating the logs behind a feature will allow on the over side to keep 0 overhead when disabling logs.

# Guide-level explanation

Now, how does it work ?

The logger works using 3 different types of logs.

- raw events
- tasks
- fork join graph

I'll start by the raw events. Since logging modify application behaviors we need to be able to log at a very low (and deterministic) cost.
Basically each thread records in memory a few bytes tag for every event happening to him.
Raw event logging is very fast, in the order of nanoseconds. Well except we need an access to the thread local storage since
each thread stores its own events. Sadly this access takes much longer in the order of 100 nanoseconds. Merging the raw events capture
in rayon will enable us to use rayon's calls to the thread local storage directly and avoid some of these extra costs.

Raw events are:

- task creation
- task end
- a link in the graph between tasks
- some additional counters being logged

This information contains everything we need but is designed to be easy and fast to write and not easy to read.
Later on when our experiment is completed we will post-process the raw events and create a `RunLog` structure
containings a set of *tasks*.

In our graphical logs every small colored rectangular block you see is a task. It contains a starting time, an ending time and links information.

Finally in order to display the logs we need some postprocessing. We build a real graph structure to compute the positions in space of all the blocks.
The positionning algorithm has been surprisingly enjoyable. It was designed to avoid overlaps and suprisingly often aligns together tasks with the same
semantics.

Note that **tasks** in `rayon-logs` are not identical to **tasks** in `rayon`. `rayon`'s tasks are further subdivised into `rayon-logs` tasks.
For example the following code:
```rust
println!("a");
rayon::join(|| println!("b"), || println!("c"));
println!("d");
```

will create four tasks *a, b, c, d* in `rayon-logs`. This has two benefits:

- you can see what happens before and after the join
- you can now have a real fork-join graph because you fork from *a* and join in *d*.

We also provide a `subgraph` function for tagging tasks with `usize`.

```
pub fn subgraph<OP, R>(tag: &'static str, work_amount: usize, op: OP) -> R
where
    OP: FnOnce() -> R,
```

This will execute the `op` closure which can be sequential (only one task) or parallel (a whole fork-join subgraph) and tag it with a `str` and an integer.
The integer typically contains the expected work amount (like the theoretical complexity) but has also been used successfully with hardware
performance counters to count the number of cache misses for example.
Once we have this information we can use it to compute speeds (since we now have work and time for each subgraph) and use this information to vary the colors
brightnesses in the svg display.

## New functions and types

I have built a preliminary version of an integration of `rayon-logs` into `rayon`.
You can find the code [here](https://github.com/wagnerf42/rayon).

This implementation is just to get the discussion started. It does seem reasonable to me but there are still some open questions.

What it does is:

- implement a storage space for the raw events logs
- change the `Registry` to include a vector of one `Storage` per thread
- change `in_registry_cold` and `in_registry_cross` to create an initial task
- change `join` to log the raw events
- provide a `subgraph` function
- provide the conversion function from raw events to tasks
- provide a `save_logs` function

All graph and svg related functions are not here to keep the changes small.

All modifications are in `rayon-core` only.
All modifications are in included in `cfg` which means you'll fall back to the standard rayon if you do not enable the `logs` feature.

All `scope` and `spawn` are not implemented yet. It shouldn't be long but I'd rather like some feedback first.

# Implementation notes

Files coming from `rayon-logs` are in the `logs` subdirectory.

- `subgraph.rs` contains the subgraph functions
- `raw_events.rs` the definitions of the raw events
- `runlog.rs` the tasks logs definitions and conversion function
- `storage.rs` the structure used for storing raw events on each thread

Then the modified files in `rayon-core` are currently:

- `join/mod.rs` for the logged join implementation
- `registry/mod.rs` for modifications to the `Registry` structure, the `in_registry` functions and the global `save_logs` function.

On the other side `rayon-logs` has been adapted to allow disconnected graphs.

## Implementing `Storage`

The `Storage` structure is a linked list of vectors (capacitated).
The idea is that most of the time you just push the event with a very little cost.
However should you log a lot of events you will have another block re-allocated for you, still in O(1) complexity (well, plus the cost of
the memory allocation).

It contains an `UnsafeCell` since threads write events to it but someone is going to read the logs and save them.
Since the vectors are never re-allocated reading a block is always safe. We could have problems
with lists pointers but I use an atomic counter for the number of blocks which is incremented only after a pointer is changed.
So, this should hopefully also be safe.

Currently logs are never erased. Calling `save_logs` several times will re-save some tasks several times.

## Implementing `join`

We changed the join function in the following way:

- we use the `in_worker` function to get access to the current thread
- we go up to the registry and use the thread index to push events in the corresponding storage
- `op_a` and `op_b` are hijacked to log a new task start at their start and a new task end at their end
- the current task is ended just before the join and restarted just after
- we tag the links in the graph between the tasks

# Unresolved questions

## Crates Dependencies

Currently I depend on the following crates. Should I remove these dependencies ?
- time (easy to remove)
- itertools (harder to remove but ok)
- serde (also easy to remove since I output logs in json)

## Svg

All graph algorithms and svg display are not included.
Is it a good idea ? This means you cannot view the svg display for logs without
installing `rayon-logs`.

Alternatively I can also remove the `RunLog` type and file and save logs in an even cruder format
to keep more processing in `rayon-logs`. If we keep the crates separated this might actually be a good idea.

## Logged Iterator

There is a convenient `LoggingIterator` in `rayon-logs` for calling `subgraph` automatically (with automatic size)
on each terminal task of a `ParallelIterator`. I could but it here but it is not completely sound (should'nt be zipped for example).
Well, since you can also easily map to call `subgraph` it might not be a necessity.

## Sharing structs with `rayon-logs`

What is the way to go for sharing structs with `rayon-logs` (avoid the type re-definition) ?
I would rather have the events types public in rayon but it could be the other way round.
Note that this problem disappears if the svg code is moved inside rayon.

## Optimizing raw events types.

We could maybe rewrite the raw events to decrease their sizes (and do more push).
I'm not sure it is a big deal though since all costs are dominated by the thread local storage.

## Saving logs.

It is the user's responsibility to not save logs while using threads. It should not crash but the graphs obtained
might be strange since saving logs takes a lot of time.
Also, should we keep a counter (atomic ? what if someones calls it in parallel ?) in each storage to count how many events have already been saved ?
This way we could call the saving function several times in a row and continuing only with the new tasks.
