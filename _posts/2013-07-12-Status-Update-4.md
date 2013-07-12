---
layout: default
---

#Weekly Status Report #4

> tl;dr ― Wrapping up debug info generation for enums and some venturing into the LLVM code base.



The last few days I have worked mostly on wrapping up support for `enums` in the `trans::debuginfo` module. This finally prepares the improved handling of memory layout I already [talked about 2 weeks ago]({{site.url}}/2013/06/28/Status-Update-2.html) for being merged  into the official master branch of Rust. These changes, along with others, are currently under (undoubtedly merciless) review by [jdm](https://github.com/jdm) in a [pull request](https://github.com/mozilla/rust/pull/7710) from which the following summary is shamelessly copied:

> This pull request includes various improvements:
>
> - Composite types (structs, tuples, boxes, etc) are now handled more cleanly by debuginfo generation. Most notably, field offsets are now extracted directly from LLVM types, as opposed to trying to reconstruct them. This leads to more stable handling of edge cases (e.g. packed structs or structs implementing drop).<br><br>
> - `debuginfo.rs` in general has seen a major cleanup. This includes better formatting, more readable variable and function names, removal of dead code, and better factoring of functionality.<br><br>
> - Handling of VariantInfo in `middle::ty` has been improved. That is, the<br>`type VariantInfo = @VariantInfo_` typedef has been replaced with explicit uses of `@VariantInfo`, and the duplicated logic for creating `VariantInfo` instances in `ty::enum_variants()` and `typeck::check::mod::check_enum_variants()` has been unified into a single constructor function. Both functions now look nicer too :)<br><br>
> - Debug info generation for enum types is now mostly supported. This includes:
>   + Good support for C-style enums. Both DWARF and gdb know how to handle them.
>   + Proper description of tuple- and struct-style enum variants as unions of structs.
>   + Proper handling of univariant enums without discriminator field.
>   + Unfortunately gdb always prints all possible interpretations of a union, so debug output of enums is verbose and unintuitive. Neither LLVM nor gdb support DWARF's `DW_TAG_variant` which allows to properly describe tagged unions. Adding support for this to LLVM seems doable. gdb however is another story. In the future we might be able to use gdb's Python scripting support to alleviate this problem. In agreement with [jdm](https://github.com/jdm) this is not a high priority for now.
> - The debuginfo test suite has been extended with 14 test files including tests for packed structs (with Drop), boxed structs, boxed vecs, vec slices, C-style enums (standalone and embedded), empty enums, tuple- and struct-style enums, and various pointer types to the above.
> 
> What is not yet included is DI support for some enum edge-cases represented as described in `trans::adt::NullablePointer`.

I also actually tried to implement support for `DW_TAG_variant` and `DW_TAG_variant_part` in LLVM and I think I had a working [proof of concept](https://github.com/michaelwoerister/rust/commit/b1eb977a0483e23fcf60cfb1b9cf60b15acc4091). However, I then could not find any evidence that gdb actually made use of these _DIEs_, so I scraped it for now.

That I was even able to implement this in a few short hours speaks for LLVM's rather nicely structured codebase (even despite their letting all locals and arguments start with a capital letter―I mean, come on, girls! Really?)

After some [changes](https://github.com/mozilla/rust/commit/2d3262ca7b94b53178daa06fa72d5427584ae842) by [nikomatsakis](https://github.com/nikomatsakis) to the way stack space is allocated for pattern bindings, the [code](https://github.com/michaelwoerister/rust/commit/fdc47f65dc443f2c7419439e44044971bb5f079e) for destructured local variables from last week also seems to work quite well. At least my [test cases so far](https://github.com/michaelwoerister/rust/blob/fdc47f65dc443f2c7419439e44044971bb5f079e/src/test/debug-info/destructured-local.rs) suggest that.

I also did some [reporting](https://github.com/mozilla/rust/issues/7715) and (hopefully) [fixing](https://github.com/mozilla/rust/issues/7712) of issues. All in all, I really enjoy working on `rustc` and with Rust and my fellow rusties. ⚘