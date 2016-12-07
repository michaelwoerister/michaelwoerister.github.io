---
layout: default
---

# Better Dependency Tracking for the Rust Compiler

The Rust compiler uses dependency tracking in order to find out what needs to
be re-compiled during incremental compilation. The model of data
dependency currently used by the compiler is less powerful than it could be
though. In this blog post I'll show what its shortcomings are and what a more
expressive model could look like.



## The Current Model

The compiler uses a *dependency graph* to track what data depends on
what other data. Each node in the graph represents some piece of data and an
edge between two nodes means that the data at the source of the edge was
needed to compute the data at the target of the edge:

```c
   MIR(foo) --> MachineCode(foo)

   ^^^ the MIR of function foo is needed to compute the machine code of it
```

This graph tells us that if the MIR of `foo` changes then we have to re-compute
the machine code for that function. And this is probably fine in most cases,
there are hardly any changes to the already pretty detailed MIR of a function
that would not require re-generating its machine code. But let's take a look at
another example:

```c
   AST(foo) --> FnSig(foo) --> TypeCheck(user_of_foo)
```

This graph tells us that the function signature of `foo` depends on the AST of
`foo` and that type checking another function (which contains a call to `foo`)
in turn depends on `foo`'s function signature. Now suppose we change the AST
in some way that does not actually affect the function signature -- like making a
parameter `mut`, something which is only meaningful within the body of a
function but has not influence on callers. The dependency tracking won't be
able to determine that the signature actually hasn't changed. It just sees that
the AST has changed and that there's an edge from the AST to the signature, so
the signature might have changed too. As a consequence, we need to re-compile
all callers of `foo` in order to be really sure that everything is up-to-date
again.

The underlying problem is that the dependency graph can only give very
conservative information about changes. An edge from A to B in the graph means
that changing the value of A might *potentially* have an influence on the value
of B. This is always correct but it might lead to many false positives for cases,
like the one above, where a change to the input of some function does not
actually change its output. In fact, the quality of dependency information
given by this model is so weak that we only get any meaningful re-use during
incremental compilation because we manually augmented it with an additional
mechanism. Somewhat unintendedly, this additional mechanism turns out to be
exactly what I'd wanted to introduce to the dependency tracking model anyway
in order to alleviate the current model's shortcomings.


## A More Powerful Model

It's a natural extension for the existing dependency tracking model to add some means
that allows to determine if a node has *actually* changed after being flagged
as having *potentially* changed. We do something like this already for the
input nodes of our dependency graph: the AST/HIR nodes and commandline arguments.
For each of those we compare if the current version has changed with respect
to the cached version and only then flag dependents as also having
(potentially) changed. In theory this can be done for any node in the graph.
All you need is some way to tell whether the current value of a node is the
same as the previous one. In the case of the AST, for each node we store a
hash value of the nodes' contents when persisting the dependency graph. But
you could also store the actual value if that happens to be cheaper than the
hash.

With this information we can implement cache invalidation "firewalls" as
[Niko](https://github.com/nikomatsakis) likes to call them: We can switch a
node's status back from *maybe changed* back to *unchanged*, thus stop cache
invalidation short at that point -- with potentially
huge implications on transitive dependents of the node. Let's take a look
again at the example from before:

```c
   AST(foo) --> FnSig(foo) --> TypeCheck(user_of_foo)
```

When the compiler discovers that the AST of `foo` has changed, it will
flag `FnSig` and `TypeCheck` as potentially out-of-date. However, in contrast
to what would happen with the current model, when it's time to re-compute the
function signature of `foo`, the compiler may discover that it is actually
exactly the same as before and propagate this information down to its
dependents, thus marking `TypeCheck(user_of_foo)` as `unchanged` again
(that is, if no other dependency of `TypeCheck(user_of_foo)` requires it to
be recomputed -- more on that some other time). In a case like the above this
can easily make the difference between having to re-compile one module and
having to re-compile every single module in the code base.


## Conclusion

I'm sure that switching to the more powerful model will result in much better
re-use rates, especially when we start to cache MIR and type-checking
information -- because what we have now won't scale well to that.
The new model will also make the compiler's implementation more
unified by not requiring additional concepts for some places, like we do
now for AST/HIR and codegen unit partitioning.

On the other hand, there will be some new implementation challenges. The
current model is so simple and conservative that we can make some assumptions
that make its implementation much easier (like the assumption that there is
no harm in doing cache invalidation in one big swoop at the beginning of the
compilation process). I'll try to write about these new challenges and
subtleties in follow-up posts, in no small part in order to get a good
understanding of them myself before jumping into an actual implementation.
<img class="blackflower" src="{{site.url}}/images/flower-black.svg"></img>
