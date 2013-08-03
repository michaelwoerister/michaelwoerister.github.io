---
layout: default
---

# Visibility Scope in Rust Debug Info
The last few days I've mostly been working on the creation of proper scoping information in Rust debug symbols. As often is the case, it soon turned out that this is a deeper, more complex topic than it looked on first sight. This post will provide a small chronology of my journey into its unexpected crevices.

<center>
<img src="http://michaelwoerister.github.io/images/hanoi.jpg" alt="Towers of hanoi. Source: http://commons.wikimedia.org/wiki/File:PSM_V26_D464_The_tower_of_hanoi.jpg"></img>
</center>



## Point of Departure
Debug info generation in `rustc` already had rudimentary support for describing visibility scopes but the implementation was fragile and missed some important features. Most notably, it could not properly describe how [variable shadowing](http://en.wikipedia.org/wiki/Variable_shadowing) in Rust works. In typical rust code, it is rather common to re-use variable names:

```rust
let result = some_function();
do_something(result.x, result.y.z);

let result = some_other_function();
do_something_else(result.a, result.b);
```

It is important to note that the name `result` will refer to two different variables here. They are stored at different locations and may even have different types. In C/C++, Java, Scala, and many other languages this would give you an `invalid re-declaration of variable` compilation error because in these languages no two local variables within the same scope can have the same name (although shadowing occurs in these languages too, e.g. local variables shadowing global ones). `GDB` too, unfortunately, thinks there should only be one variable of a given name and will always display the contents of the variable declared first in the scope, regardless of the current position in the code. The DWARF standard does not mention name shadowing. It allows for two variable declarations with the same name in the same scope and provides the `DW_AT_start_scope` attribute, the combination of which would allow to concisely describe name reuse for locals. However, LLVM does not emit `DW_AT_start_scope` attributes and `GDB` does not take notice of additional variable DIEs with equal name.

Fortunately, there is another way of properly handling this problem. For every Rust program a C-style scoped version with equivalent visibility semantics can, quite easily, be inferred. We just need to introduce an *artificial* scope every time a variable name is reused. So when a piece of code looks like this:

```rust
{
    let x = 1;
    let x = 'c'
    let x = 0.5;
}
```

we describe it in debug info as if it looked like this:

```rust
{
    let x = 1;
    {
        let x = 'c'
        {
            let x = 0.5;
        }
    }
}
```

These nested scopes would be perfectly legal in C (and `GDB` treats Rust basically as C) and they allow to achieve the desired behavior. As a side note, we actually want to create artificial scopes like the following:

```rust
+-------+
| let x | = 0;
|       +------------+
|                    |
.                    .
.                    .

```

That is, the initialization expression on the right should not let be contained in the new scope, so that (totally legal) expressions such as below show up in the debugger as expected:

```rust
let x = 1;
let x = x + 1;
```

## Implementation Attempt #1
Once it was decided that artificial scopes were the way to go, I started modifying the existing scoping info code to keep track of which variables were declared in a given block and to create artificial scope *DIEs* (Debug Information Entries) whenever a name was used more than once in the same block. This scoping information was updated on-the-fly whenever the AST-to-LLVM IR translation encountered a `let` binding.

Later, when the translation process wanted to assign the correct scope info to some statement or expression, it would take the source code position of the statement (stored by the parser in the AST) and search all scopes in the containing block, *real* and *artificial*, until it found the one containing the source code position in question.

I was *not* particularly fond of this approach for a couple of reasons:

+ It relied a lot on public functions of the `debuginfo` module to be called from the outside in the correct order---i.e. the user of the module had to make sure not to request the scope of an expression before translating all `let` bindings before that expression.
+ Searching for the correct scope by code-location seemed kind of roundabout, and relied on expression/statement source spans to rise monotonically with evaluation order. 

Yet, the approach was a natural evolution of the existing system and delivered seemingly stable results. Until macros entered the picture...

## Macro Alert!
When I tried to compile some code containing a macro, source locations suddenly were all over the place. It took me some minutes to understand why. Consider the following snippet (with line numbers):

```rust
0  macro_rules! plus_one (
1     ($e:expr) => (
2         $e 
3         + 1)
4  )
5 
6  fn main() {
7      plus_one!(41);
8  }
```

After macro expansion it looks somewhat like this:

```rust
6  fn main() {
1      (
7      41
3      + 1)
8  }
```
Notice the strange line numbers? The parser will keep code location information of expressions from within macros as they are, which makes sense: the `+ 1` from the example *is* at line 3 and not at some artificial line 7.5. But when we try to find the correct scope of an expression by its code location alone we got ourselves a problem. Semantically everything here is contained within the top-level scope of the `main()` function. However, when the `debuginfo` module is asked to find the correct scope of `+ 1` in line three, it won't know where to look for. To complicate things, the `+ 1` expression could have been expanded at any number of locations in the code. Definitely not good...

Some of this can be mitigated by some additional information the parser generates: For source spans that arise from within a macro, the call-site of the macro is stored. So in the case above, we could have transformed the line numbers 1 and 3 into 7 and would have been able to find the correct containing block.

However, take a look at the following case:

```rust
 0  macro_rules! abs (
 1    ($e:expr) => (
 2      if $e < 0 {
 3        -$e
 4      } else {
 5        $e
 6      }
 7    )
 8  )
 9
10  fn main() {
11    abs!(41);
12  }
```

expands to

```rust
    10  fn main() {
     1    (
2/11/2      if 41 < 0 {    // Note: 'if' and '< 0 {' are on line 2 
  3/11        -41          //       '41' is always on line 11
     4      } else {
    11        41
     6      }
     7    )
    12  }
```

The part substituted for `$e` will always be recorded as being on line 11, for every case. But all of them are in a different scope! Now, please find the correct scope for the expression on line 11 and no additional information. That should be pretty hard.

I had to go back to the drawing board and come up with a better solution for the whole problem.


## Implementation Attempt #2 - Precomputed Scope Map

The new method should not have any of the detriments of the former one: 

+ No reliance on code locations as these are not unique in the presence of macros.
+ The module-external interface should not have a complicated, order/state-dependent protocol.
+ More control over where new scopes are introduced (the former method also could not handle artificial scopes for e.g. bindings at the head of match statement arms).

The first point can be addressed by using AST node IDs instead of source text locations. These are always unique, even for different expansions of the same macro, and the AST structure is the definite source for the program structure and, consequently, the visibility scope structure.

The second point can be handled by not providing any state mutating public function in the `debuginfo` module. There is only one (side-effect-free) function for associating AST nodes with their correct scope DIE. The scope map that maps nodes to scopes is built internally the first time a function is touched during the translation process. From then on the scope map is immutable. This is facilitated by walking the whole AST of the function once and building the correct scope tree structure. Doing this in one step also allows to keep all related logic in one (big) function.

The last point is also solved by explicitly walking the AST, instead of trying to somehow reconstruct the needed information from various auxiliary data structures built for other parts of the translation process. This is what the former method did.

The [resulting algorithm](https://github.com/michaelwoerister/rust/blob/lexical_scopes_alt/src/librustc/middle/trans/debuginfo.rs) looks very promising. It is much easier to understand and already more full-featured than the former one. It still needs some more testing and fixing of corner cases, but the basic structure seems to allow to do so with strange hacks being needed.

Have a nice weekend! ⚘