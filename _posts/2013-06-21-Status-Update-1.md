---
layout: default
---
#Weekly Status Report #1

> tl;dr ― It's been great. Wrote some documentation, wrote quite a
> few test cases, found and fixed a bug regarding struct and tuple padding.



It seems that it's time for my first status update, so here goes <img class="blackflower" src="{{site.url}}/images/flower-black.svg"></img>

I think I got a good start working in the Rust code base. I began doing my first work package,
adding documentation to debuginfo.rs and cleaning up the code. It turned out that
[vadimcn](//github.com/vadimcn) had a [pull request](//github.com/mozilla/rust/pull/7134)
in the works that would replace direct LLVM metadata generation with calls to LLVM's
`DIBuilder`. Vadim was nice enough to get this ready before I started. Not only will this make
`rustc` more robust against future updates of LLVM, for me it also meant a significantly cleaner
`debuginfo.rs` to start my project from. So thanks again, Vadim!

Of course there has still been more than enough for me to do. I put up some
[general documentation](//github.com/michaelwoerister/rust/commit/5d5311dc74b2bce19a754538dfd3c849e8c989ed)
about the debug symbol generation process and how the module fits into the rest of the code base. I
[restructured](//github.com/michaelwoerister/rust/commit/290d35312a8c74d4652d2e8196234151f9efcabf)
the file a little to more clearly separate public and private functionality of the module. This
clean up will go on over the next three months. I hope it will be visible from the code that the
module has its own dedicated caretaker now. The whole thing is wrapped up in a
[pull request](//github.com/mozilla/rust/pull/7255).

I then went on to extending the coverage of automated debug info testing. My ongoing work on this
can be found in the [WP2 branch](//github.com/michaelwoerister/rust/tree/WP2) of my Rust fork.
So far the tests have been rather minimal and I'll try to create a solid test suite that will
provide some confidence in the debug info generation.

If it hadn't already been obvious before that this is a good idea, it certainly is now: I found a
bug within the first hour of writing tests. It was a rather subtle issue regarding the size of structs and
tuples. We created debug info that told the debugger the exact size of the struct―that is without
any padding at the end needed to ensure correct alignment when the struct is embedded within another
data structure (such as another struct, a tuple, or a vector). `gdb` had no problem handling such
ill-described structs as long as they where not embedded anywhere, so they slipped right through the
existing test cases which just checked whether a struct was correctly read directly from a local
variable.
Once I found the cause of the problem, it was
[fixed](//github.com/michaelwoerister/rust/commit/1bbfc811a6f6b22b766a6c96360a5e7ec6185ebd#L0R569)
rather quickly by always rounding the size of the struct to the next multiple of its alignment. This
is also how C's `sizeof` operator behaves (I guess, otherwise pointer arithmetic couldn't facilitate
array access).

However, as the memory layout of some Rust types can theoretically be
[rather complicated](//github.com/mozilla/rust/blob/master/src/librustc/middle/trans/adt.rs#L12)
this part of debug symbol generation will have to undergo some refactoring and improvements anyway
if we want this code work stably.

For next week, I plan to continue creating more test cases and―hopefully with the help of the
[mailing list](//mail.mozilla.org/listinfo/rust-dev)―to devise a good strategy of keeping the
debuginfo module in sync with any memory layout changes defined in some other part of the compiler.

Last but not least, I don't want to let my GSoC Mentor [Josh](//github.com/jdm/) go
unmentioned here. He has been very kind and helpful throughout. Thanks Josh ☺
