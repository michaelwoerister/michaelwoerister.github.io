---
layout: default
---

# Attempting to Use Rust's Type System for Statically Checked Dependency Tracking

In order to support incremental compilation, the Rust compiler tracks dependencies
between the pieces of data it reads and produces. It does this so it can determine
what needs to be re-computed when something in the source code changes. I have
long suspected that Rust's type system would make it possible to ensure, at
compile-time, that the dependency tracking system is used correctly, that is,
that the people writing compiler code don't forget to register data dependencies
in the appropriate places. After all, the Rust's type system, initially designed
for safe memory management, has proven to be useful in other areas too, like
preventing data races and statically checked
[fork-join parallelism](https://github.com/nikomatsakis/rayon).
In the following I'll describe how I tried to achieve similar correctness
guarantees for the dependency tracking system via rather simple and ergonomic
means. Did I succeed? Read on!



## What exactly is Dependency Tracking?

The compiler needs to record which parts of the source code affect which parts
of its output. For example, it needs to know which object files it needs to
regenerate if some function definition has been changed by the user.

This information is stored in a so-called *dependency graph*. Each node in the graph
represents some piece of input -- e.g. said function definition -- or some piece
of data the compiler has derived from these inputs -- e.g. the AST of the
function or the machine code produced for it. Whenever the compiler generates
a piece of data, it will introduce a *read-edge* from the node representing that
piece of data in the graph to all the nodes it reads while computing the
data. Consequently, if we want to know whether some computed, cached piece of
data is still valid after the input program has changed, we can take the node
corresponding to the cached data and transitively follow all of its edges to
arrive at the inputs it depends on. If any of these inputs has changed, we need
to recompute. So far so good, but can we be sure that we added all the needed
edges to the dependency graph?


## What can go wrong during dependency tracking?

The dependency tracking prototype as currently implemented in the compiler tries
to make it easy to get things right. Whenever code produces a value that
corresponds to a node in the dependency graph, we push a *task* onto a
thread-local stack that remembers which piece of data we are currently writing
to. The data we are potentially reading from is usually stored in a so-called
`DepTrackingMap`. This map knows about dependency tracking and when something
reads from it, it will insert a read-edge from the node belonging to the current
task to the node corresponding to the data we are reading. As a
result, tracking happens mostly automatically and transparently -- but problems
arise when we are holding on to data from the *DepTrackingMap* longer than
we are supposed to. Consider the following example:

```rust
let mut name_map = HashMap::new(); // <-- a regular hashmap

tcx.dep_graph.with_task(DepNode::ItemSignature(some_id), || {
    // This is fine, the HIR map will register an edge from the node
    // ItemSignature(some_id) to the node HIR(some_id) because we are
    // within the ItemSignature(some_id) task.
    let hir_item = tcx.map.expect_item(some_id);

    // But *this* is not fine at all:
    name_map.insert(some_id, hir_item.name);
});

// do something else, potentially, probably, using
// the data stored in name_map.
```

After `with_graph()` has returned, the `ItemSignature(some_id)` task
has been popped from the thread-local task stack, and trying to read from the
HIR map without a task would at least result in a runtime error.

But it's too late, we have leaked untracked data out of the system by storing
the item's name into `name_map`. We can read from `name_map` without the tracking
system ever knowing about it and thus we could generate compilation artifacts
that rely on the name being unchanged without the compiler knowing about this
dependency and thus reusing the artifact. This would probably result in a
linker error and a corrupt cache. Can we use Rust's type system, and in
particular, the borrow-checker to prevent this kind of thing?


## Preventing Data Leaks using the Type System

The problem in the example above is that the data we read from the HIR map is
used after it should not be used anymore, because we only record dependency
edges for usages within `with_task`. This sounds an awful lot like a
"use after free" problem, which is exactly what the borrow-checker was designed for
preventing. So how do we exploit this resemblance?

First, let's phrase the problem differently: We want references to tracked data to
never outlive the scope within which dependency tracking is active. This gives us
a hint that we have to somehow represent both, the tracking scope and references
to tracked data. Let's start with the scope.

The dependency tracking system, as currently implemented, already has the
so-called `DepTask` type, an RAII object one can request explicitly with
`DepGraph::in_task()` or implicitly with `DepGraph::with_task()` as done above.
When the tracking system creates an object of this type, it pushes the given
task onto the thread-local task-stack. The `DepTask` object is stored in a local
variable somewhere and, in well-known RAII fashion, will tell the dependency
tracking system that we are done writing in its destructor.

```rust
{
    let task = dep_graph.in_task(DepNode::ItemSignature(some_id));

    // some code reading tracked data ...

} // <-- read-edges from DepNode::ItemSignature(some_id)
  //     will be recorded until here
```

Now that we have this object on the stack, the compiler can reason about its
scope, which gives us the first piece of our system. The next step is to define a
type that represents references to tracked data -- we'll simply call it
`Ref`. We'll also make `Ref` a smart-pointer, so it can be used like a regular
reference, and we will endow it with some additional information for which task
we have registered a read-edge when the `Ref` was created:

```rust
struct Ref<'task, 'data, T>
    where T: 'data // the value of T must outlive the reference to it
{
    task: &'task DepTask,
    data_ref: &'data T,
}
```

As you can see, every `Ref` stores a shared reference to the `DepTask` it was
created in. This has the effect that the borrow-checker will statically ensure
that no such `Ref` can outlive the scope of the task for which we already have
correctly registered our read-edges. Let's make the data referenced by a `Ref`
accessible by implementing the `Deref` trait. This turns `Ref` into a
smart-pointer:

```rust
impl<'task, 'data, T> Deref for Ref<'task, 'data, T> {
    type Target = T;

    fn deref(&self) -> &T {
        self.data_ref
    }
}
```

Since the direct reference to the data that is returned by `deref()` cannot
outlive the `Ref` object and the `Ref` object, in turn, cannot outlive its task,
we have made sure that these direct references can't leak either. Thus, we
have effectively ensured that tracked data can only be read after the
corresponding read-edges have been added to the dependency graph and that the
data is only accessible while we are in the task that the edges were created
for.

The final piece of the puzzle is to make sure that this new setup actually gets used,
for example by making the `DepTrackingMap` only give out information if it is
passed a reference to the current task:

```rust
impl<K, V> DepTrackingMap<K, V> {

    pub fn get<'data, 'task>(&'data self,
                             task: &'task DepTask,
                             key: &K)
                          -> Ref<'task, 'data, V>
    {
        // .. register the read ...

        Ref {
            task: task,
            data: self.wrapped_map.get(key).unwrap()
        }
    }
}
```

Due to `Ref` being a smart-pointer, using this system looks almost as before:

```rust
let mut name_map = HashMap::new();

tcx.dep_graph.with_task(DepNode::ItemSignature(some_id), |task| {
    let hir_item = tcx.map.expect_item(task, some_id);
    name_map.insert(some_id, hir_item.name);
});
```

Cool! Now there is no way of getting at the data in the map that would miss
registering the read-edge, there can be no data leakage, and we are
assured of this at compile-time. Too good to be true? Well, yes, unfortunately `:)`


## Caveat: Clone and Copy

The borrow-checker might prevent us from leaking references to data but it does
not help us if someone were to just make a copy of the data and then *leak that*.
It is, after all, primarily concerned with protecting memory and not with
protecting "information". Perhaps this could be compared to it preventing
data-races but not all race conditions in general.

So, despite our laudable efforts, the example from the beginning would
still create a data leak, since the item's name is just copied into `name_map`
instead a reference to it being stored. Perfectly safe from a memory point
of view, but we would need some more protection.


## Conclusion

We did not quite succeed in our endeavor but I still think this goes to show
that Rust's type system is quite powerful when it comes to keeping tight control
over resources. Would the dependency tracking system not require preventing
information leakage in the most general sense, the simple tools above might have
done the trick -- and they might be useful in other cases where one has
different kinds of resources one needs to protect. In fact, how `Mutex`
and `RefCell` in the standard library are implemented, can be
considered a variation of this (shall we call it "compile-time RAII") and I
encourage everyone to think about how the borrow-checker can help design
APIs that enforce correct usage at compile-time and without incurring any
runtime overhead!
<img class="blackflower" src="{{site.url}}/images/flower-black.svg"></img>


#### PS.
In a real-world implementation we would not have to actually store a reference
to the task in `Ref`. The borrow-checker is only interested in the lifetime
associated with the task, and so we can replace the reference with a
`PhantomData` field that does not take up any space at runtime:

```rust
struct Ref<'task, 'data, T: 'data>
{
    task: PhantomData<&'task DepTask>,
    data_ref: &'data T,
}
```
