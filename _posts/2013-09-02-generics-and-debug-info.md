---
layout: default
---

#Generics and Debug Information in Rust
A couple of weeks ago I added support for [generics](http://static.rust-lang.org/doc/tutorial.html#generics) to <span style="font-variant: small-caps">rustc</span>'s debug info generation and found that it would be an interesting topic to write about. So, what are generics? In short, they allow program items―such as functions, structs, or traits―to be parameterized with *type variables*. That is, expressions within those items―such as locals, arguments, or fields―are declared not with a concrete type, such as `int`, but with a not-yet-concrete type designated by some name:

```rust
// T1 and T2 are type variables
struct Pair<T1, T2> {
    a: T1,
    b: T2
}
```

In order to create a value of a generic type such as the above, concrete types have to be substituted for the type variables. This is a very useful tool of abstraction, allowing to have just one `List<T>` instead of `IntLists` and `FloatLists` and `TransmogrifierLists`, to name one very common example. It is so useful that the concept of type variables can be found in nearly any [statically](http://www.stroustrup.com/) [typed](http://www.java.com) [mainstream](http://www.ecma-international.org/publications/standards/Ecma-334.htm) [and](http://dlang.org/) [not](http://www.scala-lang.org/)-[so](http://www.haskell.org)-[mainstream](http://nimrod-code.org/) [language](http://en.wikipedia.org/wiki/ML_%28programming_language%29) developed not too long ago. Quite the success story, given that ["generics is for academics only"](http://blogs.msdn.com/b/dsyme/archive/2011/03/15/net-c-generics-history-some-photos-from-feb-1999.aspx) :P



##How is generic code represented in machine code?
There are two major options how to translate generic code to low-level (virtual) machine code. The two extreme positions at each end of the spectrum, *code specialization* and *code sharing*, have a representative in C++ and Java, respectively. *Templates* (the C++ form of generics) can be viewed as a source text expansion mechanism. Templates are more restricted than C preprocessor macros, making them safe and (a bit) more sane to use―but in essence, for every configuration of concrete arguments to a template used in the source code, a specialized copy of the item's definition is created. In other words, the following code

```c++
template<typename T>
void swap(T& a, T& b) {
    T temp = a;
    a = b;
    b = temp;
}

// ...
swap(int_var1, int_var2);
swap(float_var1, float_var2);
swap(bool_var1, bool_var2);
```
will result in three separate versions of `swap` being translated, one for `T = int`, one for `T = float`, and one for `T = bool`:

```c++
void swap_int(int& a, int& b) {
    int temp = a;
    a = b;
    b = temp;
}

void swap_float(float& a, float& b) {
    float temp = a;
    a = b;
    b = temp;
}

void swap_bool(bool& a, bool& b) {
    bool temp = a;
    a = b;
    b = temp;
}

// ...
swap_int(int_var1, int_var2);
swap_float(float_var1, float_var2);
swap_bool(bool_var1, bool_var2);
```
The resulting machine code will be the same as if the template had been expanded to their specialized versions by hand. This has the advantage of templates not incurring any runtime speed penalty over non-template code. However, it also has the often criticized disadvantage of creating "bloated" executables with many copies of very similar code.

Java goes a different route here: It shares the [same byte code between all instances](http://www.angelikalanger.com/GenericsFAQ/FAQSections/TechnicalDetails.html#FAQ100) of the generic item. This is possible because nearly every value in Java is a reference and your computer doesn't much care whether it passes around a reference to an `Integer`, a `List`, or an `Object`―they are all treated the same after type checking.

This results in a lot leaner machine code but does incur some overhead as the compiler has to insert runtime type checks into the code to keep it safe. It may also rob the compiler of the chance to perform some type specific optimizations―however, given the kind of black magic the JVM regularly practices internally I wouldn't be surprised if this were alleviated some way by runtime optimizations.

C# uses a mixture of both approaches, specializing for value types and reusing code for reference types.

And Rust? <span style="font-variant: small-caps">rustc</span> performs [monomorphization](http://static.rust-lang.org/doc/tutorial.html#generics) which is a fancy term for creating specialized copies of generic items, just as C++. One important difference to C++, however, is that Rust's type system allows to do all type checking *before* expanding generic types to their specialized versions. Another (possible) difference is that <span style="font-variant: small-caps">rustc</span> and [later LLVM](http://llvm.org/docs/Passes.html#mergefunc-merge-functions) will try to merge identical functions in order to reduce code bloat. Yet, since this is a LLVM feature, <span style="font-variant: small-caps">Clang</span> can potentially do the same for C++ code. Other C++ compilers might be just as smart. So, in essence, the situation on the machine code level is the same for Rust and C++.

##Generating Debug Information for Generic Code
In order to find out what the debug information for generic code should look like we once again have to consult the [DWARF standard](http://dwarfstd.org/). The relevant section for generic functions (in version 4 of the standard) is `3.3.7 Function Template Instantiations`:

> In C++, a function template is a generic definition of a function that is instantiated differently
when called with values of different types. DWARF does not represent the generic template
definition, but does represent each instantiation.

Luckily for us, this can be applied to Rust *as is* for the reasons listed above. The rest is "just a matter of programming" (I can't find the blog post this quote is from originally but like it a lot :) The existing DI code taking care of functions had to be adapted to taking generics and type substitution into account. First, it has to perform type-substitution for a given function's signature to get rid of an [annoying ICE](https://github.com/mozilla/rust/issues/8443) (internal compiler error). Then it has to combine the information available from the AST (namely `ast::Generics` nodes) and the information from type inference (in `trans::FunctionContext::param_substs`) and generate `DW_TAG_template_type_parameter` debug information entries from it. To get this right it takes some fiddling around and handling special cases of various interactions between type parameters on traits/structs/enums, self types, and type parameters on the method/function itself. That's not particularly hard, just a bit cumbersome. [Pull request #8554](https://github.com/mozilla/rust/pull/8554) should do the job pretty well.

## Remaining Issues
The aforementioned pull request does not implement everything as demanded by the DWARF standard. Point (2) of `3.3.7 Function Template Instantiations` says:

> The subprogram entry and each of its child entries reference a template type parameter entry
in any circumstance where the template definition referenced a formal parameterized type.

In the context of Rust, the above statement means that the parameter `x` in the following function should be described as having type `T=int` instead of just type `int`:

```rust
fn generic_fun<T>(x: T) -> T {
    ...
}

fn main() {
    let s_have = Some(generic_fun(11111));
}
```

At the moment just the concrete type `int` is recorded, not what type variable it has been substituted for. This is still on the <span style="font-variant: small-caps">todo</span> list.

That does not mean, however, that you can't already inspect locals, arguments, fields, etc having generic definitions with the current debug info implementation. It just means that some information about type variables is not represented in the debug information. For the regular debugging workflow this shouldn't be much of a problem. The concrete type usually is what one is really interested in and what is needed to allow the debugger to display a variable's value. Which type variable was used to declare a local or argument can easily be read from the source code, so this is not a pressing issue while there are more important things to do.

(As a side note, while writing this post I tried to find out what it would take to implement this and it turns out that it would be quite easy for local variables and arguments. For struct and enum fields it would be a bit more work)

That's it for today! I hope you found this an interesting read. Have a good week! \<<img class="blackflower" style="padding: 0px 0px 0px 0px" src="{{site.url}}/images/flower-black.svg"></img>\>



