---
layout: default
---
# Rust Debuginfo and Unique Type Identifiers
Over the last few days I've been working on some code that creates unique type identifiers for the
types in your Rust programs. For scenarios where the intermediate LLVM code of multiple crates is
merged into one compilation unit (as is the case when link-time optimization is enabled), LLVM needs
a way of telling which type debuginfo is the same in both crates. This allows it to get rid of
duplicate data. Also, when there is a conflict in the type identifiers (i.e. two different types
have the same identifier) LLVM will abort with an assertion, an
[issue](https://github.com/mozilla/rust/issues/13681) that has been poping up a few times lately.
So I set out to create something that would solve this issue once and for all, a task that turned
out to be more difficult, but also more interesting than I anticipated.



## What we seek in a Unique Type Identifier
There are a few requirements the identifiers to be created must fulfill:

+ Equivalent types *should* result in equivalent IDs, different types *must* result in different
  IDs. I write 'should' here because in the first case, the worst that will happen is some debugging
  data redundancy, which is not desirable but also not the end of the world. The second case is
  much worse, as it may lead to corrupt debuginfo and LLVM crashes/assertions, if LLVM runs into it.

+ The identifier must be valid and unique across multiple crates, even though we do not know which
  crates this will be. <span style="font-variant: small-caps">rustc</span> already now emits
  something into the unique identifier field, but it
  is only valid within the crate being currently compiled ― which is exactly what causes the
  assertion issue mentioned above. We need something that can reliably be reconstructed from
  information that is available to the compiler when pulling in LLVM intermediate code from other
  crates.


## Simple Identifiers for Named Types
A first naive approach would be to try and take the fully qualified name of a type as its
identifier, much as one does in source code. However, this poses a few problems:

+ Through `type` declarations and re-exporting, a type might be known under multiple names.
+ Multiple types may have the same path. The following example shows a valid Rust program with two
  distinct types having the same path `::main::Foo`:

```rust
fn main() {
	{
		struct Foo { a: int }
	}

	{
		struct Foo { x: f64 }
	}
}
```
So, no luck with that approach. Fortunately, <span style="font-variant: small-caps">rustc</span>
stores metadata about each type reachable from other crates: The AST node ID (contained in the crate
metadata) of the type's definition uniquely identifies any struct, enum, etc. within the crate it is
defined in. Combined with the [Strict Version Hash](https://github.com/mozilla/rust/blob/0.10/src/librustc/back/svh.rs#L11) of the defining crate, we have a globally unique type identifier. We just
have to extract this information ― and make sure that we always take the original definition in
the case of types inlined from other crates, where multiple copies of a type definition exist.
Here is an example of such a type identifier:

```c
{7a741058d6a06b08::a1f342}
 ^^^^^^^^^^^^^^^^  ^^^^^^
    crate-hash      AST node ID as hex number
```

## Structurally Typed Types
Rust also has types without names, like tuples, pointers, and function types. These kinds of types are
considered equivalent if their components are equivalent (i.e. they are [structurally typed](http://en.wikipedia.org/wiki/Structural_typing)). This property needs to be reflected by the unique type identifier so that
two equivalent tuple types always result in the same identifier. A straightforward way of achieving
this is to simply build our type identifier from the identifiers of the component types, like in the
following example:

```c
// The tuple
(int, MyStruct, &MyStruct)

// MyStruct's type ID
{7a741058d6a06b08::a1f342}

// The tuple's type ID
{TUPLE {int} {7a741058d6a06b08::a1f342} {REF {7a741058d6a06b08::a1f342}}}
```
How the components are actually combined is not important, as long as it's unambiguous.

## Generic Types
Monomorphization, performed by the Rust compiler, is the process that creates distinct instances
of generic types for each type parameter combination used in the code. That is, there is only one
definition of `Option<T>` (and thus also only one AST node ID) but `Option<int>` and `Option<char>`
and `Option<Foo>` all designate different types which means that they also need different type
identifiers.

As you might have guessed, generics can be dealt with just the same as structural types: Just
include the identifiers of the type arguments in the identifier of the generic type instance.

```cpp
// A generic struct
struct Foo<T1, T2> { ... }

// A concrete instance of this struct
let x: Foo<Bar, int> = ...

// The type ID of Foo<Bar, int>
{STRUCT 7a741058d6a06b08::93a3fe {7a741058d6a06b08::ff312} {int}}
        ^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^
     crate-hash + node ID of Foo       type ID of Bar      type ID of int
```

With the above means at our disposal, we can easily specify a complete algorithm.

## Rust Wrinkle: Lifetime Parameters
To the Rust compiler two instances of some `Type<'a>`, i.e. with two different concrete lifetimes
substituted for `'a`, are indeed two different types, just as with regular generics. DWARF debuginfo
on the other hand does not care about lifetimes anymore. They completely vanish in the codegen
phase. As a consequence, there are N types in Rust that map to one type in DWARF. This is something
where unique type identifiers come in quite handy. A unique type identifier is unique only with
regard to DWARF type descriptions, meaning that they need not/must not take lifetime arguments into
account: All instances of a `Type<'a>` will have the same identifier, and by computing the
identifier for some type instance (which we need to do anyway), we get a nice key into a
lookup table that allows to find out whether there already is an existing DWARF type description we
can reuse. This let's us get rid of debuginfo data redundancies and probably also of some very
subtle inconsistencies in the data model, caused by the N to 1 mapping messing with LLVM's metadata
uniquing.

## No Recursive Identifiers
I tried to come up with a type constellation that would introduce cycles into the type graph that
needs to be walked while generating identifiers. Normally, types can be recursive in the sense that
they refer to themselves in their definition. Linked lists are a classic example for this:

```rust
struct ListElement {
  data: int,
  next: *ListElement  // <-- type contains its own name in definition
}
```
Dealing with this kind of recursive type definitions already requires quite a bit of
machinery in the <span style="font-variant: small-caps">rustc</span>'s debuginfo module, and for
some time during prototyping I thought the same would apply to dealing with unique type identifiers.
However, I couldn't come up with a concrete example of a type requiring a recursive identifier. For
regular generic types, one just cannot write a concrete type instance that has itself as parameter,
because that would lead to an infinite name:

```rust
struct ABC<X> { ... }

// the type name just explodes into infinity like
// 1 + 1 rabbits in a Fibonacci sequence
let x: ABC<ABC<ABC<....
```
So far so good, but Rust has the `Self` type which confused me quite a bit. How do I deal with the
following case:

```rust
struct StrangeListElement<TNext> {
  data: int,
  next: TNext
}

let list: StrangeListElement<&Self> = ... // it even kind of makes sense..
```

Fortunately, Rust does not allow this kind of thing. The consequences would have been dire. For
recursion to work, unique type identifiers would have needed some way of encoding references to
type ids that are currently being defined. The IDs themselves can't be used for that obviously so
there would have needed to be some kind of relative addressing scheme or the introduction of names for IDs
(that is, *meta-identifiers* if you want to call them that). With multiple layers of indirect
recursion and the need for identifiers to
be stable regardless of where you enter the now potentially cyclic type graph, this turns out to be
quite the mind-bender. The prospect of having to untangle this mess already had my grey cells plan
their escape through every orifice available `:)` Well, maybe it would not have been quite that bad,
but the added algorithmic complexity would no doubt have had a very real impact on implementation
complexity (and thus bug count and maintainability) and perhaps also on runtime cost, so I'm glad
we dodged that particular bullet.

## Conclusion
I'm still working on getting this implemented in <span style="font-variant: small-caps">rustc</span>
but the [results](https://github.com/michaelwoerister/rust/tree/unique_type_id) look promising so
far. If you have any comments, please post them to the corresponding [/r/rust thread](). <img class="blackflower" src="{{site.url}}/images/flower-black.svg"></img>
