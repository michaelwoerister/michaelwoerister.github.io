---
layout: default
---

#Weekly Status Report #3

> tl;dr ― Enum support is in the works.

A rather short update this week. I've been working on supporting the various kinds of enums, starting with c-style ones (that is, their values just consist of a discriminant and have no fields). This worked out rather nicely, as they have a direct equivalent in C and LLVM's `DIBuilder`, DWARF, and `gdb` have good support for handling them.



I then went on to tuple-style enum variants, like the one below:

```rust
enum ABC {
	Variant1(int, char, ~str),
	Variant2(bool, bool)
}
```
I thought, it would be a good idea to describe them as unions (as in C `union`) because that's how they are represented in memory: Every value of such a variant is just a tuple, with the discriminant value prepended as the first element. This way, one can determine which variant this is by looking at the first element (which is in the same position for all variants) and thus determine what the rest of the value looks like (how many elements there are and which types do they have).

The problem is that `gdb` won't do this. These safely *tagged unions*, like we also have in Pascal or Ada for example, do not exist in C/C++ and `gdb` does not support them either (at least I could not find anything about it). `gdb` always prints all possible interpretations of a union-typed value, leading to rather unintelligable output. One can still find out what the actual value is because the discriminant shows the right value in all versions but it's far from ideal usability-wise.

I don't know yet what to do about this. The three possible options I see:
+ Try to extend `gdb` to support tagged unions. This sounds very much out of scope for the GSoC project. I don't even know whether DWARF supports them well (though I guess it does).
+ Try to find a DWARF description that tricks `gdb` into printing nicer output. Enums are very similar in memory layout to a single-inheritance one-level class hierarchy with an empty abstract base class, they just have a tag value instead of the vtable-pointer. Maybe this can be exploited somehow. I don't know yet.
+ Do nothing about it; which is my favored option for the moment, as I don't want to get distracted from implementing more important things.

While I tried to cleanly support special cases (like enums with just one variant which do *not* have a discriminant value) I got blocked by some [functionality not yet being implemented](//github.com/mozilla/rust/issues/7527) in the compiler. I followed [Josh's](//github.com/jdm) suggestion and tried to solve the issue, which I [seemingly did](//github.com/mozilla/rust/pull/7557). Now I am waiting for the pull request to be merged into master. After that I'll also implement struct-like enum variants (i.e. where each element has a name).

In the meantime I tried to be productive and started implementing correct debug info generation for destructuring variable bindings like the following:
```rust
let (a, (b, c)) = (1, (2.0, 'a'));
```
This turned out to be rather simple in theory, using the handy `pat_utils::pat_bindings()` function. However, I still get wrong debug information for the general case and I can't quite explain why. I'll need to take a look at the LLVM IR generated from the new implementation (next week).

Well, that's it for today. Not so short after all <img class="blackflower" src="{{site.url}}/images/flower-black.svg"></img>
