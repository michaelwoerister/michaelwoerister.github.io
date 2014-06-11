---
layout: default
---

# Programming Portfolio
This is a selection of some things I did programming-wise over the last decade. It reflects my interests in game programming, program architectures for soft-realtime and graphics applications, multi-threading/parallelism, and programming languages in general.

## Rust (2013)
This summer I've been working on the compiler of Mozilla's new programming language [Rust](//www.rust-lang.org), in the context of a Google Summer of Code project. My main task has been the implementation of debug symbol generation, so Rust programs can be debugged using GDB or LLDB. It's very interesting to work on a bootstrapping compiler for a still evolving language, or in other words, debug the debugging code in a compiler that compiles itself `:)` In the future I'd like to keep contributing in this area, as time permits. The summer project has brought some great progress to the debug info module but obviously there is still work to do.

* [Blog: Conclusion Report](//michaelwoerister.github.io/2013/09/27/what-you-call-the-present.html)
* [Blog: Generics and Debug Information in Rust](//michaelwoerister.github.io/2013/09/02/generics-and-debug-info.html)
* [Blog: Visibility Scope in Rust Debug Info](//michaelwoerister.github.io/2013/08/03/visibility-scopes.html)

## Phase (2013)
Experimental execution model for games, allowing for parallelization and simpler reasoning about game logic. Proof of concept implementation in C++. Very similar to what John Carmack later described in his QuakeCon 2013 keynote when talking about game programming in Haskell.

* [github.com/michaelwoerister/phase](//github.com/michaelwoerister/phase)

## A Caching System for a dependency-aware Scene Graph (2011-2013)
Master thesis written at the [VRVis](www.vrvis.at). Describes and evaluates a system that allows optimized scene graph rendering using data dependency analysis, incremental computation, and compiler techniques. Prototype implemented in C# within a large rendering framework. Continued work after the thesis has been published as a paper at the [High-Performance Graphics 2013](//highperformancegraphics.org/) conference.

* [Thesis Poster](//www.cg.tuwien.ac.at/research/publications/2012/Woerister_2012_ACS/Woerister_2012_ACS-Poster.pdf)
* [Paper](//www.vrvis.at/publications/PB-VRVis-2013-018)
* [Thesis](//www.cg.tuwien.ac.at/research/publications/2012/Woerister_2012_ACS/)


## Redd & Blou (2010-2012)
<a href="{{site.url}}/images/portfolio/rnb0.png">
  <img width="200px" src="{{site.url}}/images/portfolio/rnb0_thumb.jpg"></img>
</a>
<a href="{{site.url}}/images/portfolio/rnb1.png">
  <img width="200px" src="{{site.url}}/images/portfolio/rnb1_thumb.jpg"></img>
</a>
<a href="{{site.url}}/images/portfolio/rnb2.png">
  <img width="200px" src="{{site.url}}/images/portfolio/rnb2_thumb.jpg"></img>
</a>
<a href="{{site.url}}/images/portfolio/rnb3.png">
  <img width="200px" src="{{site.url}}/images/portfolio/rnb3_thumb.jpg"></img>
</a>
<a href="{{site.url}}/images/portfolio/rnb_editor.png">
  <img width="200px" src="{{site.url}}/images/portfolio/rnb_editor_thumb.jpg"></img>
</a>

A puzzle platformer with two simultaniously controlled characters and a Braid-inspired rewind feature. Programmed in C#, using XNA, including a map editor with undo/redo functionality. Never got finished due to lack of spare time and a clear vision of what the final game should play and look like. Artwork by [Martin Wörister](//miascugh.multimediaart.at/).

* [Demo version (Windows) including some 10 playable levels](//www.dropbox.com/s/g0p7z5dqsti21ru/ReddAndBlouDemo.zip)
* [XNA 4 Redistributable (required)](//www.microsoft.com/en-us/download/details.aspx?id=20914)

## Engine of Evermore (2008-2009)
Component-based 2D game engine written in C# (also running on Linux using Mono), including map editor with undo/redo functionality, file system abstraction layer, optimized sprite sheet packing. Never finished, but very educational to work on.

* [Source Code on code.google.com](//code.google.com/p/engineofevermore/)

## Platinum.PropertyGrid (2008-2009)
A replacement for .Net's [System.Windows.Forms.PropertyGrid](//msdn.microsoft.com/en-us/library/system.windows.forms.propertygrid%28v=vs.110%29.aspx) with support for realtime feedback and transactional property manipulation protocol. Developed for the editor of Engine of Evermore.

* [Source Code on code.google.com](//code.google.com/p/platinum-propertygrid/)

## TGB-Box2D Integration (2008)
For my bachelor thesis I integrated the Box2D physics engine into the Torque Game Builder runtime engine and level editor. Has been used by others within several commercial projects, afaik.

* [Source Code on code.google.com](//code.google.com/p/tgb-box2d-integration/)
* [Thesis](//tgb-box2d-integration.googlecode.com/files/Integrating%20Box2D%20into%20TGB%201.0.pdf)


## Chasing (2006)
<img src="{{site.url}}/images/portfolio/chasing.png"></img>

Pac Man-esque game with a ninja turtle fighting zombies in a dungeon (obviously). Created using Torque Game Builder (which I was working on as a contractor at the time). Features 2-player split screen mode.

* [Game (Windows)](//www.dropbox.com/s/sivl08tlffk5v6o/ChasingDemo.zip)

## Lander Reloaded (2001)
<img src="{{site.url}}/images/portfolio/lander.jpg"></img>

A lunar lander game programmed during summer holiday in Delphi, featuring many planets and moons of our solar system, including both (!) Earth moons.

* [Game (Windows)](//www.dropbox.com/s/b4nk0l2ba1qpos6/LanderReloaded.zip)
* [//www.caiman.us/scripts/fw/f1760.html](//www.caiman.us/scripts/fw/f1760.html)
