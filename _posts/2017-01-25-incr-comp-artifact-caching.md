---
layout: default
---

# Incremental Computation Model Refresher

I originally planned to describe the improved dependency tracking algorithm for
the Rust's incremental compilation as the follow-up to the
[last post](http://michaelwoerister.github.io/2016/12/07/better-tracking.html).
However, while doing a draft of that, I kept getting confused about cached
results and their exact relationship to nodes in the dependency graph. I think
it's useful to write down what my mental model for incremental computation in
the compiler is, especially since it's slightly different than what's described
in the [incr. comp. alpha version blog post](https://blog.rust-lang.org/2016/09/08/incremental.html)
and also in the [incr. comp. RFC](https://github.com/rust-lang/rfcs/blob/master/text/1298-incremental-compilation.md).



## The Model
In order to get a clearer picture, I'll try to sort out what the central
concepts here are and how they relate to each other. My mental model looks
about like this:

 * There are `computations` in the compiler, each of which has a result
 * There are `cache entries` which can be loaded from disk
 * There is the `dependency graph` consisting of nodes and edges between them

Let's define those terms in more detail:

### Computations

The model views the compilation process as a graph of computations, where
each `computation` is the application of some function `f` to a set of inputs,
which in turn are `computations` themselves. Each computation is a node in this
graph, and if one computation takes the result of another computation as input,
then there is a direct edge between those two nodes. The roots of the
graph are the input values of the compilation process (e.g. source files,
command-line args, etc), which can be viewed as trivial computations, yielding
their result without need for inputs.
Each computation has a result, although that result sometimes is just
success or no success (e.g. borrow checking). Here's an illustration of how a
compilation session could be modeled this way:

```rust

                lib.rs
                  |
                  v
            parse(lib.rs)
             |         |
             v         v
     gen_hir(fn main)  gen_hir(struct Foo)
      |        |                 |
      |        v                 v
      |  type_info(fn main) type_info(struct Foo)
      |           |          |      |
      |           v          v      |
      |          infer(fn main)     |
      |           |                 |
      v           v                 |
    gen_mir(fn main) <--------------+
       |       |
       |       v
       |     borrow_check(fn main)
       v
    translate(cgu-0)
          |
          v
    llvm_compile(cgu-0.ll)
             |
             v
       link(main.exe)

```

This is of course a simplified example, but in theory the whole compilation
process can be modeled like that.


### Cache Entries
The next term on our list is `cache entry`, i.e. the stuff that's stored in the
incremental compilation cache directory. In theory this could be anything that
can be loaded from disk in order to help not re-compute something during
re-compilation, but here I'll define it more strictly: a cache entry is the
serialized result value of one computation node from the computation graph. In
other words, each computation in the compilation can have its result cached and
we can use the ID of the computation to see if there is something cached for
that computation. Note, however, that we don't *require* that there is a
cache entry for the result of every computation.

Given a computation graph and a cache, it's easy to see how this enables
speeding up re-computing the graph: Whenever I reach a computation `X`, I
can take a look if I already have the result of `X` in the cache. But how do we
know whether the cached result of `X` is still valid. That's where the next
term comes into play.

### Dependency Graph
The purpose of the `dependency graph` is cache invalidation. We want to be able
to ask it "given that these inputs have changed, which cache entries need to
be purged?"
With this in mind,
it might be a bit surprising that there is a separate concept of a `dependency
graph`. Isn't the computation graph already the dependency graph? The answer is
yes and no. Yes: it's a perfect dependency graph, and no: it's not the
dependency graph that is actually used by the compiler. The reason is tracking
granularity. The number of computations (each of which, conceptually, must have
the properties of a pure function application) would be too high to keep track
of efficiently and often there would not even be a theoretical benefit in
doing so.
Consequently, we try to construct a graph that still fulfills the goal of
proper cache invalidation while not being more fine-grained than it
needs to be. This is possible because it is safe to "conflate" multiple
computation nodes into one dependency node. For example, consider two
computations `A` and `B`, with their inputs `I1`, `I2`, and `I3`:

```rust

    I1 ----+
           |
           v
    I2 --> A

    I3 --> B

```

Our dependency graph must ensure that `A`'s cache entry is invalidated if
either `I1` or `I2` changes, and that `B`'s cache entry is invalidated if `I3`
changes. Now let's say, we replace the nodes `A` and `B` with one new node
`A&B` and associate this node with both `A`'s and `B`'s cache entries:

```rust

    I1 -----+
            |
            v
    I2 --> A&B
            ^
            |
    I3 -----+

```

We still can give the same guarantees: If `I1` or `I2` changes, `A`s
cache entry is purged (because both cache entries are) and if `I3` changes
then `B`'s cache entry is purged (because, again, both are). So we have
effectively traded tracking overhead for getting more false positives during
cache invalidation. (Note that there are even cases where conflation will net
a pure win, where we get rid of unnecessary granularity).

So, we can finally define that the dependency graph is an approximation of the
computation graph such that:

 * each dependency graph node corresponds to a set of computation nodes, and
 * if computation node `A` is in the set of dependency node `A'`, and computation
   node `B` is in the set of dependency node `B'`, and there is an edge between
   `A` and `B`, then there is also an edge between `A'` and `B'`.

I'll also add the restriction that these sets are disjoint, i.e. that every
computation node corresponds to exactly one dependency node. That's what one
gets anyway, if one starts out with a computation graph and starts merging
nodes.

### Summary
So we have the computation graph, each node of which corresponds to zero or one
cache entries. And we have the dependency graph, each node of which corresponds
to one or more computation nodes. Thus we can link each dependency node to all
corresponding cache entries (which we need to during during cache invalidation)
and vice versa (which we need to do during dependency tracking).


```rust

  dep-nodes           D1               D2              D3
                     /  \              |             / |  \
  computations     C1    C2            C3          C4  C5  C6
                    |                  |                   |
  cache entries    E1                  E2                  E4

```


### Some Examples
Often there is a one to one to one correlation between dep-node, computation, and
cache entry:

```rust

 dep-nodes:    |   Mir(fn main)   |    Mir(fn foo)    |    Mir(fn bar)    |

 computations: | gen_mir(fn main) |  gen_mir(fn foo)  |  gen_mir(fn bar)  |

 cache:        |   Mir(fn main)   |    Mir(fn foo)    |    Mir(fn bar)    |

```

But there might also be different configurations, with conflation and no
caching, for example:

```rust

 dep-nodes:    |                TraitSelect(Debug)                |

 computations: | <i32 as Debug> | <f64 as Debug> | <Foo as Debug> |

 cache:        |   (no entry)   |   (no entry)   |   (no entry)   |

```


## But the RFC has procedures and write-edges and whatnot!
The incremental compilation RFC describes dependency tracking a bit differently:

* There are "procedures" which push "procedure nodes" onto a stack, and
* when a procedure reads some piece of data, a "read edge" is added from the
  top-most procedure node to the "data node", and
* when the procedure writes some piece of data, a "write edge" is added from the
  procedure node to the data node.

This sounds quite different, but it's actually equivalent. There is no actual
difference between "procedure nodes" and "data nodes" (and implementation in
the compiler also never has made this kind of distinction).
It is just a convenient way of mapping the
compiler's "passes that read and produce data" execution model to a
dependency graph. And there has always really only been one kind of edge in the
dependency graph: a "write edge" is just a recorded as a reverse "read edge".

In order to see that the two models are equivalent, we can provide a mapping
from one to the other:

* A `procedure node` is replaced by a `computation node` that as result just
  has a collection of all the values the `procedure node` has `read edges` to.
* Each `data node` is replaced by a `computation node` that computes its value
  from that data in the connected `procedure nodes`.

I prefer the computation graph model to the procedures-and-data graph model,
since it only has one kind of node and one kind of edge instead of two of both.


## What about other Incremental Computation model like Adapton?

There's quite a bit of existing research on incremental computation. One very
interesting model is [Adapton](http://adapton.org). It looks like it maps well
to our needs and indeed, after reading the papers, I think Nominal Adapton's
`demanded computation graph` is pretty much the same as the computation graph
as I described it above (where a `ref` in Adapation is what I called a trivial
computation and a `thunk` would be a regular computation). That would be pretty
cool because the Adapton people have put lots of effort into proving things
about the model.

## Conclusion

With the relationship between dependency graph nodes and cache entries clarified
it should be easier to describe the new cache invalidation algorithm. It will be
interesting to compare it more closely to the Adapton model. I'm sure we can
learn something from them.

<img class="blackflower" src="{{site.url}}/images/flower-black.svg" />
