---
layout: default
---
# Mozilla Contract
I'm happy to tell you that I'll be working on Rust's debuginfo support in the context of a contract with Mozilla. I have been to able fix some bugs and add a few features after my [GSoC](//www.google-melange.com/) project. However, having the contract means that I'll be able to consistently spend time on Rust―two days per week over the coming months, to be precise. Which is a fabulous thing, needless to say `:)`

Apart from fixing [existing and emerging bugs](//github.com/mozilla/rust/issues?labels=A-debuginfo&state=open), there'll be two major areas that I'll work on:



1. Making LLDB a first-class debugger for Rust. Currently, we only have auto-tests for GDB, meaning that if debugging Rust in LLDB works, it's accidental more or less. This will change soon and will help shield LLDB support against regressions. Especially for Mac OS users this should be good news.

2. Both GDB and LLDB support Python extensions that allow to control how values in the debugger are displayed. I'll use this feature to finally have the debuggers print values in Rust syntax. So no more strange enums (`{ {Alt1, 0xdeadbeef}, {Alt1, 0xdead, 0xbeef} }`) and real rust pointer sigils.

In conclusion, these are exciting times for me and I also hope good times for Rust's debuginfo support. Thanks, Mozilla, for making this happen and thanks especially to [Brian](//github.com/brson) who has been super helpful over the last weeks!

<img class="blackflower" src="{{site.url}}/images/flower-black.svg">
