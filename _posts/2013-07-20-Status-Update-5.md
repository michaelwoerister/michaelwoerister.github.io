---
layout: default
---

#Weekly Status Report #5

Not many words from me this time. This week I worked on:

+ Getting the big [Work Package 4](https://github.com/mozilla/rust/pull/7710) pull request ready
  for merging and extending it with some fixes (e.g. supporting the new memory layout of unique vecs).
+ [Added support](https://github.com/michaelwoerister/rust/commit/6d1a52db6d96c35efe4bd1ae9b2bf5edaed819e6)
  for debug info about locals declared with destructuring patterns. This works very stably already.
+ Fixed function argument handling which has been broken for a while now. Unfortunately, there is
  still an issue with function argument destructuring. I don't know really what's going on here 
  because the code should essentially be the same as with local variables. This will be the first
  thing to look into next week.  
+ Spent some spare time doing general refactoring and naming cleanup work on 
  [`syntax::ast`](https://github.com/mozilla/rust/pull/7903) and 
  [`rustc::middle::trans`](https://github.com/mozilla/rust/pull/7848).



Have a nice weekend! <img class="blackflower" src="{{site.url}}/images/flower-black.svg"></img>
