---
layout: post
title: "The MMTk post that needs a better name"
author:
- Octave Larose
---

This is a blog post about the garbage collection framework [MMTk](https://github.com/mmtk/mmtk-core) and my experience implementing it into our own Rust-based intepreters.

## what am i reading?

This is part of a series of blog posts relating my experience pushing the performance of programming language interpreters written in Rust. For added context, read the start of [my first blog post]({% post_url 2024-05-29-to-do-inlining %}).

In short: we optimize AST (Abstract Syntax Tree) and BC (Bytecode) Rust-written implementations of a Smalltalk-based research language called [SOM](http://som-st.github.io/), in hopes of getting them fast enough to meaningfully compare them with other SOM implementations. The ultimate goal is seeing how far we can push the AST's performance, the BC being mostly to be a meaningful point of comparison (which means its performance needs to be similarly pushed).

As a general rule, all my changes to the original interpreter (that led to speedups + don't need code cleanups) are present [here](https://github.com/OctaveLarose/som-rs/tree/best)...

...and benchmark results are obtained using [Rebench](https://github.com/smarr/ReBench), then I get performance increase/decrease numbers and cool graphs using [RebenchDB](https://github.com/smarr/ReBenchDB). In fact, you can check out RebenchDB in action and all of my results for yourself [here](https://rebench.stefan-marr.de/som-rs/), where you can also admire the stupid names I give my git commits to amuse myself.

## what have i been doing?
It's been ages since the previous blog post (six months, so 7 years or so in dog years), so a lot of things. Most of which are described in hundreds of commits.

The performance of both our AST and BC interpreters is much, much better: something like 100% (yeah!) for the AST and 70% for the BC. I could show them here but I'd rather hold off on addressing them for now, since there's potentially very interesting results for my own research I'd like to expand on in the future (and/or check if I'm messing something up somewhere before drawing conclusions). But at this present moment, AST and BC performance are very similar.

Both interpreters have been worked on extensively and even undertook major reworks, which are explained by this blog post's subject: we now do garbage collection using MMTk, instead of relying on reference counting.

I can motivate this whole blog post by showing the performance we got from adopting tracing GC using MMTk, though!

![bde96ba315406117f87c3246310f6e990df557d4..8aa557da960c81b13750d1cdb3fb9e8928693bc2](/assets/mmtk/total_mmtk_perf.png)

*Double* the speed, for both our AST and BC intepreters. As the ancient nugget of wisdom goes: "good shit".

Though this data can easily be misleading! Importantly: this needed *major* interpreter reworks to function, so a *lot* of changes were made and not all of them are strictly related to MMTk itself or tracing GC. What do you do when you're getting annoyed after a lot of GC debugging? You procrastinate by working on other features in your interpreter[^procrastination]. As a random example, I realized that I'd been using [`Vec::split_off`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.split_off) in a very common case instead of the better-suited [`Vec::drain`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.drain). That's a fun fact for you.

I should mention that the majority of such theoretically unrelated changes were *made possible* by changing our pointer representation to accommodate for tracing GC, though, so it is at least related.

But what did we change, and why?

## reference counting: the existing, naive approach
Our interpreters manage a lot of data that we need to be able to reference somehow. Rust offers several pointer types, be it standard references with `&T` (pointers which abide by Rust's safety guarantees), ugly raw pointers `*const T` (bad boys which disregard safety guarantees), or smart pointers, most notably `Box<T>`, `Rc<T>` and `RefCell<T>`. We were using smart pointers a lot since they also tend to *own* the data they reference, which tends to be a lot more convenient than trying to pass references around everywhere.

The basic case is a `Box<T>`: it's just a pointer to the heap, prettified and wrapped by Rust. OK, say in our interpreters any pointer to a `Class` object will be a `Box<Class>`. Instances of this class need a `Class` pointer to read static variables from it and - oh wait, `Box<T>` uniquely owns the data, so these instances can't all own the same `Box<T>`. So `Rc<T>` ("Rc" standing for *reference counting*) sounds like our friend: data can be referenced in several places, since we keep track of its users through a counter. And we also want to be able to write to those static variables, so we need mutable access to `T`, which is what `RefCell<T>` allows. So the final pointer type we choose: `Rc<RefCell<T>>`. Very nice.

This is very convenient: we can access and modify pesky types like `Class` from many different places, Rust manages all the memory for us and is happy, and we're happy because Rust is happy and automatically manages all the memory for us and because Rust is happy. Everyone and everything is happy.

Oh wait, except the runtime performance. Every single time we want to access e.g. a `Class`, we need to A) check that it's not already borrowed by somebody else, B) increment the reference count by 1: and every time we drop a `Rc<RefCell<T>>` after using it, we need to decrement the reference count by 1. So pointer access is *slow*, and we do a lot of it.

So reference counting is slow. What can we do instead? *Tracing garbage collection*, which is where MMTk comes in.

## tracing garbage collection
Tracing garbage collection is automatic memory management, like reference counting, which looks through all our objects to determine which ones should be discarded. Our objects are all a big graph, and GC *traces* it to see which objects are *reachable*: if you can no longer be found in the graph, you're no longer in use and in the garbage you go.

Reference counting has to check whether or not to discard an object everytime we stop using a reference to it. Tracing GC just waits until we run out of memory, in which case we stop the world and trace the object graph[^stop-the-world].

Added benefit, and a major one: we manage our own heap, not Rust. This is harder

## what's mmtk then?

The [MMTk](https://www.mmtk.io/) project has been around since 2004. Its goal is simple: tracing garbage collection is notoriously hard, so many would benefit greatly from a GC framework that does a lot of the heavy lifting. Through its API and its bindings, you can implement any of several *plans*(garbage collection algorithms - mark and sweep, semispace, )

It's got sturdy bindings for [OpenJDK](https://github.com/mmtk/mmtk-openjdk), [V8](https://github.com/mmtk/mmtk-v8) and some for [Julia](https://github.com/mmtk/mmtk-julia) are actively in development. It clearly works with much larger virtual machines than mine.

My reasoning behind choosing MMTk for `som-rs` was simple: I don't want to have to worry about GC too much, since I've got many other things I'd like to focus on. This did not account for the fact that implementing MMTk took me forever (oops).

I'm not in need of a crazy well performing GC, just a good enough one for my interpreters not to be much slower than they could be. So my approach was first to get a mark-and-sweep GC working, then moving on to a semispace collector

## allocations and pointers

## "oh no there's a problem"
talk about bugs i had

## "why are there always problems"

And that's why I'm grateful I don't handle the GC logic at least.

## where are the lifetimes?

My GC implementation is functional, but breaks free from a *lot* of Rust's safety guarantees: it's raw pointer city. There's many blog posts on garbage collection out there that describe smarter approaches than mine: you can read about Saoirse's [shifgrethor](https://without.boats/blog/shifgrethor-iii/), Core Dumped's [Emacs Lisp VM](https://coredumped.dev/2022/04/11/implementing-a-safe-garbage-collector-in-rust/) or kyren's [gc-arena](https://kyju.org/blog/rust-safe-garbage-collection/).

An obvious distinction between my work and theirs is that they make clever use of lifetimes, while my pointers are essentially always raw. I think it's a real shame my own code doesn't make smarter use of lifetimes. It was already a major hurdle to get my interpreters working correctly with tracing GC, and I'm not sure strict lifetimes would have helped me catch many of the nasty bugs I encountered. Maybe, though, so it's definitely future work.

## in conclusion
MMTk is good
You have my code as a reference
wooo

---

[^procrastination]: Annoying, I know, since it makes potentially interesting data not so usable, and I *might* write a more thorough analysis of each experiment of mine in the future if I can make it interesting. And look, you yourself also procrastinate by implementing new features instead of fixing bugs. We are kindred spirits, you and I.
[^stop-the-world]: Some GCs don't stop the world, but that's a whole other level of complexity so we don't do it. I am but one little guy.