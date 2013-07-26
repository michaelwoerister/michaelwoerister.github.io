---
layout: default
---

#Iterator Blocks for Rust

Lately, I've been thinking about what it would take to implement C#-inspired 
[iterator blocks](http://msdn.microsoft.com/en-us/library/vstudio/dscyy5s0.aspx) for
[Rust](www.rust-lang.org)―naturally something that has been 
[discussed before](https://github.com/mozilla/rust/issues/7746)―and decided to try to bring some
order into my thought process by means of this blog.

<center>
<img width="400px" src="http://michaelwoerister.github.io/images/leighton-an-italian-lady.jpg"></img>
</center>

*Iterator blocks* (in C# parlance) or *generators* (Python) are what is commonly associated with a
`yield` statement or derivations thereof. The idea behind them is to allow the programmer to write
something that looks like code typically found in the context of internal iteration, but then
automatically transform this code into something that has the properties of an *external iterator*.



##Why would one want to do that?

Bob Nystrom has a fine [article](http://journal.stuffwithstuff.com/2013/01/13/iteration-inside-and-out/)
on external vs internal iteration and their various advantages and disadvantages (also interesting:
[Yield: Mainstream Delimited Continuations](http://www.cs.indiana.edu/~sabry/papers/yield.pdf)). 
The gist of it is that internal iterators for complex (e.g. tree- or graph-like) data structures are
often easier to write because one can use the callstack to keep track of the iteration state 
involved. On the other hand external iterators have some nice properties:

+ They are first-class values one can store and pass around (you get an *iterator object*)
+ They are driven by the caller (pulling out one item at a time) which allows to 
  + consume only part of the iterated sequence and
  + interleave several sequences from different iterators (e.g. *zipping* two list)

*Iterator blocks* allow to have both advantages at the same time:

+ Their code looks pretty much the same as with internal iteration
+ The compiler transforms this code into a class/object/type implementing the interface of an
  external iterator
+ The generated iterator object can often be implemented very efficiently via a simple
  [FSM](http://en.wikipedia.org/wiki/Finite-state_machine)

So they are, hands down, a good thing. However, `yield` means slightly different things in C#, Ruby,
Javascript, Python, F#, Sather, and CLU. And it will certainly mean something slightly different
from all of the above in Rust too. So over the next few weeks I plan to explore with some depth what
the specific circumstances of an iterator block implementation for Rust are. This will probably 
include the following topics:

+ Supported features―and possible patent issues `ಠ_ಠ`
+ Possible syntax and integration with the rest of the language
+ Implementation strategies
+ How to handle:
  + destructors
  + borrowed pointers
  + closures

Of course I'll try to also discuss each of these topics on the
[Rust mailing list](https://mail.mozilla.org/listinfo/rust-dev). No use in soliloquizing about 
something that needs broad acceptance by the community. In the best case scenario this might even
turn into a proof-of-concept implementation some time in the future (no promises made though).  
I'd already be happy to have a few interested readers. ⚘
